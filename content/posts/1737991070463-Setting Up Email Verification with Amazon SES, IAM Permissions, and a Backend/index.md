---
title: "Setting Up Email Verification with Amazon SES, IAM Permissions, and a Backend"
date: 2025-01-27
draft: false
description: "Email verification cloud setup details"
tags: ["SES", "IAM"]
---

## Brief Introduction

Amazon Simple Email Service (SES) is a scalable, reliable way to send emails. This guide documented how I configured Amazon SES with the SDK, IAM permissions, and a Node.js backend.

---

## Verify Your Domain or Email Address in SES

To send emails using SES, first verify your domain or email address:

1. **Go to the SES Console**:
   - Navigate to **Verified Identities** and click **Create Identity**.

2. **Choose Domain or Email Address**:
   - For a domain, add the TXT, MX, and CNAME records to your DNS as specified.
   - For an email address, verify it by clicking the link sent to your inbox.

---
## Request Production Access
SES accounts are initially in the sandbox environment, which restricts email sending. To lift these restrictions:

1. Go to the **Account Details** section in the SES console.
2. Click on **Request Production Access**.
![alt text](<Screenshot 2025-01-27 at 4.09.54 PM.png>)

---

## Create an IAM User with SES Permissions

To securely interact with SES, create an IAM user:

1. **Go to the IAM Console**:
   - Click on **Users** and then **Add Users**.

2. **Set User Details**:
   - Enter a username, e.g., `SES-User`.

3. **Attach SES Permissions**:
   - Add the **AmazonSESFullAccess** policy to the user.
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ses:*"
            ],
            "Resource": "*"
        }
    ]
}
```

1. **Generate Access Keys**:
   - Save the **Access Key ID** and **Secret Access Key** for use in your backend.

---

## Set Up the Database

For managing users and email verification, create the following `users` table:

### SQL Table Schema

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(31) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    hashed_password VARCHAR(64) NOT NULL,
    email_verified BOOLEAN DEFAULT false,
    verification_token VARCHAR(64) NULL,
    verification_token_expiry DATETIME NULL,
    reset_token VARCHAR(64) NULL,
    reset_token_expiry DATETIME NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

This schema supports email verification and password reset functionality.

---

## Your Backend Code for Email Verification

Below is your backend implementation for handling user registration, sending verification emails, and verifying the user.

### Backend Code

```javascript
// Send verification email using Amazon SES
const verificationLink = `https://quantifyjiujitsu.com/verify-email?token=${verificationToken}`;

// Create the SES email parameters
const params = {
  Source: 'no-reply@quantifyjiujitsu.com', // SES verified email
  Destination: {
    ToAddresses: [email], // Recipient email
  },
  Message: {
    Subject: {
      Data: 'Email Verification - Quantify Jiu-Jitsu',
    },
    Body: {
      Html: {
        Data: `<p>Hi ${name},</p><p>Please verify your email by clicking the link below:</p><p><a href="${verificationLink}">Verify Email</a></p>`
      },
    },
  },
};

try {
  // Send the email via SES
  const command = new SendEmailCommand(params);
  const data = await client.send(command);
  console.log('Email sent successfully:');

  // Respond after successful registration and email sent
  res.status(201).json({ message: 'User registered successfully! Check your email to verify your account.' });
} catch (error) {
  console.error('Error sending verification email:', error);
  res.status(500).json({ error: 'Failed to send verification email' });
}
```

This code handles sending verification emails and ensures the user receives a link to confirm their email address.

---

## Test the Integration

1. **Register a User**:
   - Use a REST client (e.g., Postman) to send a `POST` request to the registration endpoint with a JSON body containing `name`, `email`, and `password`.

2. **Check the Email**:
   - Verify the email sent via SES and click the link provided.
![alt text](<Screenshot 2025-01-27 at 10.04.22 PM.png>)
1. **Verify the User in the Database**:
   - After clicking the verification link, confirm that the `email_verified` column in the database updates to `true`.

