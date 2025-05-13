## Setting up Razorpay in Next.js

### Setting context
#### user model
```typescript
export interface IUser extends Document {
    email: string;
    password: string;
    role: "user" | "admin"
    createdAt: Date
    updatedAt: Date
}
```
#### order model
```typescript
interface PopulatedUser {
    _id: mongoose.Types.ObjectId;
    email: string;
}

interface PopulatedProduct {
    _id: mongoose.Types.ObjectId;
    name: string;
    imageUrl: string;
}

export interface IOrder extends Document {
    userId: mongoose.Types.ObjectId | PopulatedUser;
    productId: mongoose.Types.ObjectId | PopulatedProduct;
    variant: ImageVariant;
    razorpayOrderId: string;
    razorpayPaymentId?: string;
    amount: number;
    status: "pending" | "completed" | "failed";
    downloadUrl?: string;
    previewUrl?: string;
    createdAt?: Date;
    updatedAt?: Date;
}
```
---

### api/orders/route.ts
```typescript
import { authOptions } from "@/lib/auth"
import connectDb from "@/lib/db";
import Order from "@/models/order.model";
import { getServerSession } from "next-auth"
import { NextResponse } from "next/server";
import Razorpay from "razorpay"

const razorpay = new Razorpay({
    key_id: process.env.razorpay_id,
    key_secret: process.env.razorpay_secret
})


export async function POST(req: Request) {
    try {
        const session = await getServerSession(authOptions);
        if (!session) {
            return NextResponse.json({
                error: "Unauthorized"
            }, { status: 401 })
        }

        const { productId, variant } = await req.json();
        await connectDb();
        const order = await razorpay.orders.create({
            amount: Math.round(variant.price * 100),
            currency: "INR",
            receipt: `receipt-${Date.now()}`,
            notes: {
                productId: productId.toString()
            }
        })

        const newOrder = await Order.create({
            userId: session.user.id,
            productId,
            variant,
            razorpayOrderId: order.id,
            amount: Math.round(variant.price * 100),
            status: "pending"
        }) 

        return NextResponse.json({
            orderId: order.id,
            amount: order.amount,
            currency: order.currency,
            dbOrderId: newOrder._id
        })
    } catch (error) {
        console.log(error);
        return NextResponse.json({
            error: "Could not create order!"
        }, { status: 500 })
    }
}
```
### api/webhook/razorpay/route.ts
```typescript
import { NextRequest, NextResponse } from "next/server";
import crypto from "crypto"
import connectDb from "@/lib/db";
import Order from "@/models/order.model";
import nodemailer from "nodemailer";


export const POST = async (req: NextRequest) => {
    try {
        const body = await req.text();
        const signature = req.headers.get("x-razorpay-signature");

        const expectedSignature = crypto
            .createHmac("sha256", process.env.razorpay_secret as string)
            .update(body)
            .digest('hex');

        if (signature !== expectedSignature) {
            return NextResponse.json({
                error: "Invalid signature"
            }, { status: 400 })
        }
        //all this above done to check that this is a valid request done by the legit razorpay

        const event = JSON.parse(body);
        await connectDb();

        //handling 1 event
        if (event.event === "payment.captured") {
            const payment = event.payload.payment.entity
            const order = await Order.findOneAndUpdate({
                razorpayOrderId: payment.order_id
            }, {
                razorpayPaymentId: payment.id,
                status: "completed"
            }).populate([
                { path: "productId", select: "name" },
                { path: "userId", select: "email" }
            ])

            if (order) {
                const transporter = nodemailer.createTransport({
                    service: "sandbox.smtp.mailtrap.io",
                    port: 2525,
                    auth: {
                        user: process.env.mailtrap_username,
                        pass: process.env.mailtrap_password
                    }
                })

                await transporter.sendMail({
                    from: "hiaayush30@gmail.com",
                    to: (order.userId as {email:string}).email,
                    subject: "Order completed",
                    text: `Your order ${(order.productId as {name:string}).name} has been successfully placed`
                })

                return NextResponse.json({
                    message: "success"
                })
            }
        }
    } catch (error) {
        console.log(error);
        return NextResponse.json({
            error: "Something went wrong"
        }, { status: 500 })
    }
}
```
### src/components/RazorpayPayment.tsx
```typescript
"use client"
import Script from 'next/script';
import React, { useState } from 'react';

const RazorpayPayment = () => {
    const [amount, setAmount] = useState('');

    const handleAmountChange = (event) => {
        setAmount(event.target.value);
    };

    const payNow = async () => {
        if (!amount) {
            alert('Please enter the amount.');
            return;
        }

        try {
            // Create order by calling the server endpoint
            const response = await fetch('/api/orders', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    amount: parseFloat(amount) * 100, // Convert to paise
                    productId: "6820fbe5a27ccf66c205008b",
                    variant: {
                        price: amount,
                        type:"POTRAIT",
                        lisence:"personal"
                    },
                    notes: {},
                }),
            });
            console.log(response)
            const order = await response.json();
            console.log("ye rha order",order)

            if (order && order.orderId) {
                const options = {
                    key: process.env.NEXT_PUBLIC_razorpay_id, // Use environment variable for key_id
                    amount: order.amount, // Amount from the backend order
                    currency: order.currency,
                    name: 'Your Company Name',
                    description: 'Secure Payment',
                    order_id: order.orderId, // Use the order ID from the backend
                    callback_url: `${window.location.origin}/dashboard?success=true`, // Use dynamic success URL
                    prefill: {
                        name: '', // You can prefill user details if available
                        email: '',
                        contact: '',
                    },
                    theme: {
                        color: '#3366FF',
                    },
                //     handler: async function (response) {
                //         // Verify the payment signature on the server
                //         const verificationResponse = await fetch('/api/webhook/razorpay', {
                //             method: 'POST',
                //             headers: {
                //                 'Content-Type': 'application/json',
                //             },
                //             body: JSON.stringify({
                //                 razorpay_order_id: response.razorpay_order_id,
                //                 razorpay_payment_id: response.razorpay_payment_id,
                //                 razorpay_signature: response.razorpay_signature,
                //             }),
                //         });

                //         const verificationData = await verificationResponse.json();

                //         if (verificationData.success) {
                //             alert('Payment successful! Order ID: ' + response.razorpay_order_id);
                //             // Redirect to a success page or update UI
                //             window.location.href = `/payment-success?order_id=${response.razorpay_order_id}&payment_id=${response.razorpay_payment_id}`;
                //         } else {
                //             alert('Payment failed: ' + verificationData.error);
                //             // Handle payment failure
                //         }
                //     },
                };

                const rzp = new window.Razorpay(options);
                rzp.open();
            } else {
                alert('Failed to create order.');
            }
        } catch (error) {
            console.error('Error creating order:', error);
            alert('An error occurred while processing your request.');
        }
    };

    return (
        <div>
            <h1>Razorpay Payment Gateway Integration</h1>
            <form id="payment-form">
                <label htmlFor="amount">Amount:</label>
                <input
                    type="number"
                    id="amount"
                    name="amount"
                    value={amount}
                    onChange={handleAmountChange}
                    required
                />
                <button type="button" onClick={payNow}>
                    Pay Now
                </button>
            </form>
            <Script src="https://checkout.razorpay.com/v1/checkout.js"></Script>
        </div>
    );
};

export default RazorpayPayment;
```