Alright, let’s make this super simple! If you’re working on a Next.js project and need to integrate Razorpay for payments, just follow these steps, copy-paste the code, and you’re done. No fluff, just what you need

## Step 1: Install Required Packages
First, install the Razorpay Node.js SDK and other necessary packages. In your project directory, run:
```
npm install razorpay @types/razorpay axios
```

## Step 2: Get API Keys from Razorpay
[Get api keys from here](https://dashboard.razorpay.com/app/website-app-settings/api-keys)
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jvl8k48lcjvfjzeco62l.png)

Store your Razorpay keys in a `.env` file at the root of your project
```
NEXT_PUBLIC_RAZORPAY_KEY_ID = YOUR_RAZORPAY_KEY_ID
RAZORPAY_KEY_ID = YOUR_RAZORPAY_KEY_ID
RAZORPAY_KEY_SECRET = YOUR_RAZORPAY_KEY_SECRET
```

## Step 4: Create an Order API Route
Next.js has a built-in API route system, making it easy to handle backend logic. Let’s create an API route for Razorpay to generate orders.

Create a new API route `/api/createOrder/route.ts`

```
import { NextRequest, NextResponse } from "next/server";
import Razorpay from "razorpay";

const key_id = process.env.RAZORPAY_KEY_ID as string;
const key_secret = process.env.RAZORPAY_KEY_SECRET as string;

if (!key_id || !key_secret) {
    throw new Error("Razorpay keys are missing");
}

const razorpay = new Razorpay({
    key_id,
    key_secret
})

export type OrderBody = {
    amount: number;
    currency: string;
}

export async function POST(request: NextRequest) {
    try {

        const { amount, currency }: OrderBody = await request.json();
        if (!amount) {
            return NextResponse.json({ message: `Amount is required` }, { status: 400 })
        }

        const options = {
            amount,
            currency: currency || "INR",
            receipt: `receipt#${Date.now()}`,
        }

        const order = await razorpay.orders.create(options);
        console.log("Order Created Successfully");

        return NextResponse.json({ orderId: order.id }, { status: 200 })

    } catch (error) {
        return NextResponse.json({ message: "Server Error", error }, { status: 500 })
    }
}
```

## Step 5: Create a function for generating OrderId
```
import axios from "axios";

export async function createOrderId(amount: number, currency: string) {
    try {
        const response = await axios.post("/api/createOrder", {
            amount: amount * 100, // Convert to paise
            currency: "INR",
        });

        console.log("Order Response:", response.data);
        return response.data.orderId;
    } catch (error) {
        console.error(error);
        throw new Error("Failed to create order");
    }
}
```

## Step 6: Create a Checkout Button
Create a reusable component `/components/BuyButton.tsx` for the Razorpay checkout button. Copy the below code:

```
"use client";

import axios from "axios";
import React from "react";
import Script from "next/script";
import { createOrderId } from "@/utils/createOrderId";

export default function PurchaseButton() {
  const [isLoading, setIsLoading] = React.useState(false);

  const handlePayment = async () => {
    setIsLoading(true);
    const price = 100; // Replace with dynamic price
    try {
      const orderId: string = await createOrderId(price, "INR");
      const options = {
        key: process.env.NEXT_PUBLIC_RAZORPAY_KEY_ID,
        amount: price * 100,
        currency: "INR",
        name: "YOUR_COMPANY_NAME", // Replace with dynamic company name
        description: "Payment for your order", // Replace with dynamic order description
        order_id: orderId,
        handler: async function (response: any) {
          try {
            const paymentResponse = await axios.post("/api/verifyOrder", {
              razorpay_order_id: orderId,
              razorpay_payment_id: response.razorpay_payment_id,
              razorpay_signature: response.razorpay_signature,
            });

            alert("Payment Successful!");
            console.log(paymentResponse.data);
          } catch (error) {
            alert("Payment verification failed. Please contact support.");
            console.error(error);
          }
        },
        prefill: {
          name: "YOUR_NAME", // Replace with dynamic user data
          email: "YOUR_EMAIL", // Replace with dynamic user data
        },
        theme: {
          color: "#3399cc",
        },
      };

      const razorpay = new (window as any).Razorpay(options);
      razorpay.on("payment.failed", function (response: any) {
        alert("Payment failed");
        console.error(response.error);
      });
      razorpay.open();
    } catch (error) {
      alert("Payment failed. Please try again.");
      console.error(error);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <>
      <button
        className="bg-emerald-700 text-white font-semibold px-4 py-2 rounded-xl hover:bg-emerald-600 transition-all duration-300 hover:shadow-lg hover:scale-105"
        onClick={handlePayment}
        disabled={isLoading}
      >
        {isLoading ? "Processing..." : "Buy Now"}
      </button>
      <Script
        id="razorpay-checkout-js"
        src="https://checkout.razorpay.com/v1/checkout.js"
      />
    </>
  );
}

```

## Step 8: Verify Payment
For added security, you can verify the payment signature server-side. Create a new API route `/api/verifyOrder/route.ts`

```
import { NextRequest, NextResponse } from "next/server";
import crypto from "crypto";

export interface VerifyBody {
    razorpay_order_id: string;
    razorpay_payment_id: string;
    razorpay_signature: string
};

export async function POST(request: NextRequest) {
    try {
        const { razorpay_order_id, razorpay_payment_id, razorpay_signature }: VerifyBody = await request.json();

        if (!razorpay_order_id || !razorpay_payment_id || !razorpay_signature) {
            return NextResponse.json({ error: "Missing required parameters", success: false }, { status: 400 })
        }
        
        const secret = process.env.RAZORPAY_KEY_SECRET as string
        if (!secret) { return NextResponse.json({ error: "Razorpay secret not found" }, { status: 400 }) }

        const HMAC = crypto.createHmac("sha256", secret)
        HMAC.update(`${razorpay_order_id}|${razorpay_payment_id}`)
        const generatedSignature = HMAC.digest("hex")

        if (generatedSignature === razorpay_signature) {
            return NextResponse.json({ message: "Payment verified successfully", success: true })
        } else {
            return NextResponse.json({ error: "Invalid signature", success: false }, { status: 400 })
        }
    } catch (error) {
        return NextResponse.json({ error: "An error occurred", success: false }, { status: 500 })
    }
}
```

Here is how you might create a products page and add checkout flow:
checkout this Github repository where I have created complete payment flow on a products page: [github/hanuchaudhary/RazorpayIntegration](https://github.com/hanuchaudhary/RazorpayIntegration)

## Done!
That’s it! You’ve:

- Installed the required dependencies.
- Added Razorpay keys to `.env`.
- Created two POST API routes: `/api/createOrder/route.ts` and `/api/verifyOrder/route.ts`.
- Built a reusable button component.
- Added the Razorpay script.
For extra features like webhooks or subscriptions, check out the [Razorpay Documentation](https://razorpay.com/docs/#home-payments).

## Conclusion

Integrating Razorpay with Next.js 14/15 is straightforward and efficient. By following these steps, you can securely accept payments in your application. For advanced features like subscriptions or webhooks, refer to the Razorpay Documentation.

_Thanks for reading this blog! Don’t forget to leave a comment💬!_
