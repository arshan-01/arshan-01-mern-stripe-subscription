# mern-stripe-subscription

### Note : Please install all required packages accordingly 


# Backend 

### .env

```

# Stripe Keys
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET_KEY=

# Subscription Plan Price IDs from stripe dashboard
PRICE_OSCE_PLAN_3=
PRICE_FULL_PACKAGE_3=
PRICE_OSCE_PLAN_12=
PRICE_FULL_PACKAGE_12=
PRICE_PASS_GUARANTEE_12=

```

### app.js
```
import dotenv from 'dotenv';
dotenv.config();
import express, { urlencoded } from "express";
import cors from "cors";
import cookieParser from "cookie-parser";
import userRouter from "./routes/user.routes.js";
import paymentRouter from "./routes/payment.routes.js";

const app = express();

app.use(cors({
    origin: "*", // Allow all origins (for testing; pass array of origins to allow)
    methods: "GET,POST,PUT,DELETE,PATCH",
    credentials: true, // If sending cookies/auth headers
    allowedHeaders: ["Content-Type", "Authorization"]
}));

// Handle preflight requests manually (important for file uploads)
app.options('*', cors());

// only use the raw bodyParser for webhooks
app.use((req, res, next) => {
    if (req.originalUrl === '/api/v1/webhook') {
        next();
    } else { 
        express.json({limit: '5mb'})(req, res, next);
    }
});

app.use(express.urlencoded({ extended: true, limit: '5mb' }))
app.use(urlencoded({ extended: true }));
app.use(cookieParser());


app.use("/api/v1", paymentRouter);

export { app };


```
### payment.routes.js
```
import express from 'express';
import {
    subscribe,
    success,
    cancel,
    customerPortal,
    webhook
} from '../controllers/payment.controller.js';

const router = express.Router();

router.get('/subscribe', subscribe);
router.get('/success', success);
router.get('/cancel', cancel);
router.get('/customers/:customerId', customerPortal);
router.post('/webhook', express.raw({ type: 'application/json' }), webhook);

export default router;

```
### /src/utils/FormateDate.js
```
export const convertToTimestamptz = (unixTimestamp) => {
    if (!unixTimestamp || typeof unixTimestamp !== "number") {
        throw new Error("Invalid timestamp provided");
    }
    
    return new Date(unixTimestamp * 1000).toISOString(); // Converts to UTC ISO format
}

```


### payment.controller.js

```
import dotenv from 'dotenv';
dotenv.config();
import Stripe from 'stripe';
import { cancelSubscription, insertSubscription } from '../utils/insert-subscription.utils.js';
import { convertToTimestamptz } from '../utils/FormateDate.js';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

const priceMap = {
    "osce-plan-3": process.env.PRICE_OSCE_PLAN_3,
    "full-package-3": process.env.PRICE_FULL_PACKAGE_3,
    "osce-plan-12": process.env.PRICE_OSCE_PLAN_12,
    "full-package-12": process.env.PRICE_FULL_PACKAGE_12,
    "pass-guarantee-12": process.env.PRICE_PASS_GUARANTEE_12
};

export const subscribe = async (req, res) => {
    const { plan, userId } = req.query;

    if (!plan || !priceMap[plan]) {
        return res.status(400).json({ error: 'Invalid subscription plan' });
    }

    // Define plans that should allow coupon codes
    const plansWithCoupons = ["full-package-3", "full-package-12"]; // Only these will have coupon fields

    try {
        const session = await stripe.checkout.sessions.create({
            mode: 'subscription',
            line_items: [{ price: priceMap[plan], quantity: 1 }],
            allow_promotion_codes: plansWithCoupons.includes(plan), // Only enable for selected plans
            success_url: `${req.headers.origin}/checkout-success`,
            cancel_url: `${req.headers.origin}/checkout-cancelled`,
            metadata: {
                userId: userId ? userId.toString() : '', // Ensure userId is a string
                plan : priceMap[plan]
            },
        });

        res.json({ url: session.url });
    } catch (error) {
        console.error('Stripe error:', error);
        res.status(500).json({ error: error.message });
    }
};

export const success = async (req, res) => {
    res.send('Subscribed successfully');
};

export const cancel = (req, res) => {
    res.redirect('/');
};

export const customerPortal = async (req, res) => {
    try {
        const portalSession = await stripe.billingPortal.sessions.create({
            customer: req.params.customerId,
            return_url: `${process.env.BASE_URL}/`,
        });

        res.json({ url: portalSession.url });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
};

// Store subscriptions in progress with a map using customer ID as the key
const subscriptionsInProgress = new Map();

export const webhook = async (req, res) => {
    const sig = req.headers['stripe-signature'];
    let event;

    try {
        event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET_KEY);
    } catch (err) {
        return res.status(400).json({ error: `Webhook Error: ${err.message}` });
    }

    try {
        const eventData = event.data.object;
        let customerId = null;

        // Extract the customer ID based on event type
        if (event.type === 'checkout.session.completed') {
            customerId = eventData.customer;
        } else if (['invoice.paid', 'customer.subscription.updated', 'customer.subscription.deleted'].includes(event.type)) {
            customerId = eventData.customer;
        }

        if (!customerId) {
            return res.json({ received: true });
        }

        // Initialize subscription object if it doesn't exist for this customer
        if (!subscriptionsInProgress.has(customerId)) {
            subscriptionsInProgress.set(customerId, {
                data: {},
                webhooks: {
                    checkout_session: false,
                    invoice_paid: false,
                    customer_subscription_updated: false,
                    customer_subscription_deleted: false
                }
            });
        }

        const subscription = subscriptionsInProgress.get(customerId);

        // Process the event based on its type
        switch (event.type) {
            case 'checkout.session.completed':
                const session = eventData;
                subscription.data = {
                    userId: session.metadata.userId,
                    plan : session.metadata.plan,
                    amount_total: session.amount_total / 100,
                    customer: session?.customer,
                    customer_email: session.customer_details?.email,
                    customer_name: session.customer_details?.name,
                    customer_phone: session.customer_details?.phone,
                    tax_exempt: session.customer_details?.tax_exempt,
                    tax_ids: session.customer_details?.tax_ids,
                    currency: session.currency,
                    payment_status: session.payment_status,
                    expires_at: convertToTimestamptz(session.expires_at),
                };
                subscription.webhooks.checkout_session = true;
                break;

            case 'invoice.paid':
                const invoice = eventData;
                subscription.data = {
                    ...subscription.data,
                    hosted_invoice_url: invoice.hosted_invoice_url,
                    invoice_pdf: invoice.invoice_pdf,
                    invoice_number: invoice.number,
                };
                subscription.webhooks.invoice_paid = true;
                break;

            case 'customer.subscription.updated':
                const updatedSubscription = eventData;
                subscription.data = {
                    ...subscription.data,
                    status: updatedSubscription.status,
                    used_tokens: 0,
                    total_tokens: 15,
                    current_period_start: convertToTimestamptz(updatedSubscription.current_period_start),
                    current_period_end: convertToTimestamptz(updatedSubscription.current_period_end),
                };
                subscription.webhooks.customer_subscription_updated = true;
                break;

            case 'customer.subscription.deleted':
                subscription.data = {
                    ...subscription.data,
                    customer : eventData.customer,
                    status: eventData.status,
                    current_period_end: convertToTimestamptz(eventData.current_period_end),
                };
                subscription.webhooks.customer_subscription_deleted = true;
                break;

            default:
                console.log(`Unhandled event type ${event.type}`);
        }

        // Check conditions before inserting subscription
if (!subscription.webhooks.customer_subscription_deleted) {
    const otherWebhooks = Object.entries(subscription.webhooks)
        .filter(([key]) => key !== "customer_subscription_deleted")
        .every(([, value]) => value === true);

    if (otherWebhooks) {
        await insertSubscription(subscription.data);
        console.log("🚀 ~ webhook ~ subscription.data:", subscription.data)
        subscriptionsInProgress.delete(customerId);
    }
    console.log(`Subscription inserted for customer: ${customerId}`);
} else {
    // Subscription was deleted, skip insertion
    await cancelSubscription(subscription.data);
    subscriptionsInProgress.delete(customerId);
    console.log(`Skipping insertion for deleted subscription: ${customerId}`);
}


        return res.json({ received: true });
    } catch (error) {
        console.error('Error processing webhook:', error);
        return res.json({ received: true, error: error.message });
    }
};



```



# Frontend


```
import React, { useState } from 'react';
import Navbar from '../components/common/Navbar';
import { CheckCircle, Tag } from 'lucide-react';
import PlanSlug from '../utils/PlanSlug';
import axios from 'axios';
import { toast } from 'sonner';

const Pricing = () => {
    const [isAnnual, setIsAnnual] = useState(false);

    const pricingPlans = {
        termly: [
            {
                title: "The OSCE plan",
                price: 35,
                monthlyPrice: (35 / 3).toFixed(1),
                "plan-slug": PlanSlug("The OSCE plan", 3),
                features: [
                    "Station specific OSCE scenarios",
                    "60 hours of OSCE bot access"
                ]
            },
            {
                title: "The Full Package",
                price: 45,
                monthlyPrice: (45 / 3).toFixed(1),
                "plan-slug": PlanSlug("The Full Package", 3),
                hasDiscount: true,
                discount: 20, // 20% discount for Termly
                features: [
                    "Everything in OSCE",
                    "MLA + Clinical Bank",
                    "SAQ question bank",
                    "Pre-clinical",
                    "Data interpretation",
                    "Question generation",
                    "Tutor Bot"
                ]
            }
        ],
        annual: [
            {
                title: "The OSCE plan",
                price: 50,
                oldPrice: 8.75, // Old price for Annual
                monthlyPrice: (50 / 12).toFixed(1),
                "plan-slug": PlanSlug("The OSCE plan", 12),
                features: [
                    "Station specific OSCE scenarios",
                    "60 hours of OSCE bot access"
                ]
            },
            {
                title: "The Full Package",
                price: 65,
                oldPrice: 11.25, // Old price for Annual
                monthlyPrice: (65 / 12).toFixed(1),
                "plan-slug": PlanSlug("The Full Package", 12),
                hasDiscount: true,
                discount: 30, // 30% discount for Annual
                features: [
                    "Everything in OSCE",
                    "MLA + Clinical Bank",
                    "SAQ question bank",
                    "Pre-clinical",
                    "Data interpretation",
                    "Question generation",
                    "Tutor Bot"
                ]
            },
            {
                title: "The Pass Guarantee",
                price: 1280,
                showTotalOnly: true,
                "plan-slug": PlanSlug("The Pass Guarantee", 12),
                features: [
                    "Everything in The Full Package",
                    "Guaranteed pass of this academic year",
                    "1-1 Tutoring from a top decile student"
                ]
            }
        ]
    };

    const getCurrentPlans = () => {
        return isAnnual ? pricingPlans.annual : pricingPlans.termly;
    };

    // Function to handle the subscription
    const handleSubscription = async (planSlug) => {
        try {
            toast.success("You are being redirected to the payment gateway");
            // Send the plan slug to the backend via an Axios POST request
            const response = await axios.get(`${process.env.REACT_APP_BACKEND_URL}/subscribe?plan=${planSlug}`);

            if (response.status === 200) {
                // Handle success (maybe show a success message to the user)
                window.location.href = response.data.url; // Redirect user to Stripe checkout page
            }
        } catch (error) {
            console.error("Error with subscription:", error);
            // Handle error (maybe show an alert or message to the user)
        }
    };

    return (
        <div className="">
            <Navbar />
            <div className="flex flex-col items-center justify-center h-screen mt-[500px] lg:mt-24 2xl:mt-32">
                <div className="font-semibold text-[24px] lg:text-[36px] text-[#52525B] text-center">
                    <p>So confident</p>
                    <p>we can even guarantee you pass.</p>
                </div>
                <div className="flex w-[224px] p-8 font-bold h-[50px] items-center justify-center bg-[#3CC8A1] rounded-[8px] text-white gap-x-8">
                    <p
                        className={`hover:bg-white/20 px-4 py-2 cursor-pointer rounded ${!isAnnual ? 'bg-white/20' : ''}`}
                        onClick={() => setIsAnnual(false)}
                    >
                        Termly
                    </p>
                    <p
                        className={`hover:bg-white/20 px-4 py-2 cursor-pointer rounded ${isAnnual ? 'bg-white/20' : ''}`}
                        onClick={() => setIsAnnual(true)}
                    >
                        Annual
                    </p>
                </div>
                <div className="text-[16px] font-medium text-[#71717A] mt-2">
                    <p>Save with an annual plan</p>
                </div>

                <div>
                    <div className="flex flex-col lg:flex-row items-center justify-center gap-x-5">
                        {getCurrentPlans().map((plan, index) => (
                            <div key={index} className="mt-5 mb-8 relative transition hover:shadow-greenBlur rounded-[16px]">
                                <div className="h-[500px] lg:h-[590px] w-[270px] lg:w-[310px] border-[1px] border-[#3CC8A1] rounded-[16px]">
                                    <div className="p-8 bg-[#3CC8A1] text-white text-center rounded-tr-[14px] rounded-tl-[14px]">
                                        <h3 className="text-xl font-semibold mb-2">{plan.title}</h3>
                                        <div className="font-bold">
                                            {plan.oldPrice && (
                                                <span className="text-2xl text-[#D4D4D8] line-through mr-2">
                                                    £{plan.oldPrice}
                                                </span>
                                            )}
                                            <span className="text-4xl">
                                                £{plan.showTotalOnly ? plan.price : plan.monthlyPrice}
                                                <span className="text-lg font-normal">
                                                    /{plan.showTotalOnly ? 'year' : 'month'}
                                                </span>
                                            </span>
                                        </div>
                                        {plan.hasDiscount && (
                                            <div className="mt-3 inline-flex items-center bg-white/20 rounded-full px-3 py-1">
                                                <Tag size={14} className="mr-2" />
                                                <span className="text-sm">
                                                    {`Save up to ${plan.discount}% ${isAnnual ? 'annually' : 'termly'}`}
                                                </span>
                                            </div>
                                        )}
                                    </div>
                                    <div className={`p-8 space-y-2 ${plan.title === "The OSCE plan" ? 'text-[16px] lg:text-[18px]' : 'text-[14px] lg:text-[16px]'}`}>
                                        <div className="space-y-4">
                                            {plan.features.map((feature, featureIndex) => (
                                                <div key={featureIndex} className="flex items-start">
                                                    <CheckCircle className="text-[#FF9741] flex-shrink-0 mt-1" size={18} />
                                                    <span className="ml-3 text-gray-700">{feature}</span>
                                                </div>
                                            ))}
                                        </div>
                                    </div>

                                    <div className="absolute bottom-5 left-1/2 transform -translate-x-1/2">
                                        <button
                                            onClick={() => handleSubscription(plan["plan-slug"])}  // Call subscription handler
                                            className="text-[16px] rounded-[8px] font-semibold text-[#3CC8A1] bg-transparent border-[1px] border-[#3CC8A1] hover:bg-[#3CC8A1] hover:text-white w-[232px] h-[40px] transition-all duration-200">
                                            Get Access
                                        </button>
                                    </div>
                                </div>
                            </div>
                        ))}
                    </div>
                </div>
            </div>
        </div>
    );
};

export default Pricing;


```
#### /src/utils/PlanSlug.js
```
const planMap = {
    "The OSCE plan": { 3: "osce-plan-3", 12: "osce-plan-12" },
    "The Full Package": { 3: "full-package-3", 12: "full-package-12" }, // ✅ Added missing comma
    "The Pass Guarantee": { 12: "pass-guarantee-12" }
};

const PlanSlug = (plan, mode) => planMap[plan]?.[mode] || "invalid-plan";

export default PlanSlug;


```

### src/pages/CheckoutSuccess.jsx

```
import React from 'react';
import { Link } from 'react-router-dom';

function CheckoutSuccess() {
  return (
    <div className="h-screen flex justify-center items-center bg-gray-100">
      <div className="bg-white p-6 rounded-lg shadow-md text-center max-w-md">
        <svg viewBox="0 0 24 24" className="text-green-600 w-16 h-16 mx-auto my-6">
          <path fill="currentColor"
            d="M12,0A12,12,0,1,0,24,12,12.014,12.014,0,0,0,12,0Zm6.927,8.2-6.845,9.289a1.011,1.011,0,0,1-1.43.188L5.764,13.769a1,1,0,1,1,1.25-1.562l4.076,3.261,6.227-8.451A1,1,0,1,1,18.927,8.2Z">
          </path>
        </svg>
        <h3 className="md:text-2xl text-base text-gray-900 font-semibold">Subscription Successful!</h3>
        <p className="text-gray-600 my-2">Thank you for completing your subscription payment. You're now subscribed to your selected plan.</p>
        <p>Enjoy your premium content!</p>
        <div className="py-10">
          <Link to="/" className="px-12 bg-indigo-600 hover:bg-indigo-500 text-white font-semibold py-3 rounded-lg">
            Manage Subscription
          </Link>
        </div>
      </div>
    </div>
  );
}

export default CheckoutSuccess;


```

#### src/pages/CheckoutCancel.jsx

```
import React from 'react';
import { Link } from 'react-router-dom';

function CheckoutCancel() {
  return (
    <div className="h-screen flex justify-center items-center bg-gray-100">
      <div className="bg-white p-6 rounded-lg shadow-md text-center max-w-md">
        <svg viewBox="0 0 24 24" className="text-red-600 w-16 h-16 mx-auto my-6">
          <path fill="currentColor"
            d="M12 2a10 10 0 1 0 10 10A10 10 0 0 0 12 2Zm1 14.93V17a1 1 0 1 1-2 0v-1.07a6.84 6.84 0 0 1-4.6-4.6H7a1 1 0 1 1 0-2H6.4A6.84 6.84 0 0 1 11 6.07V5a1 1 0 1 1 2 0v1.07a6.84 6.84 0 0 1 4.6 4.6H17a1 1 0 1 1 0 2h.6A6.84 6.84 0 0 1 13 15.93Z">
          </path>
        </svg>
        <h3 className="md:text-2xl text-base text-gray-900 font-semibold">Subscription Payment Cancelled</h3>
        <p className="text-gray-600 my-2">Your subscription payment has been cancelled. Please try again later or select another plan.</p>
        <div className="py-10">
          <Link to="/" className="px-12 bg-red-600 hover:bg-red-500 text-white font-semibold py-3 rounded-lg">
            Go to Home
          </Link>
        </div>
      </div>
    </div>
  );
}

export default CheckoutCancel;

```
