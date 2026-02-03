# India-Focused Instant Worker Matching App (Android-First MVP)

## 1. App Architecture

### High-Level Components
- **Mobile App (Flutter)**
  - Worker app + Hirer app in a single codebase (role-based UI).
  - OTP login, onboarding, document verification, training, availability, booking, payments.
- **Backend (Firebase-first, scalable to Node.js)**
  - **Auth**: Firebase Auth (Phone OTP).
  - **Database**: Firestore for realtime availability + structured data.
  - **Storage**: Firebase Storage for Aadhaar/photo uploads.
  - **Functions**: Cloud Functions for verification workflow, matching, payments, payouts.
- **Payments**
  - **Razorpay** (UPI, cards, wallets). Webhooks handled by Cloud Functions.
  - In-app wallet ledger for hirers and workers.
- **Admin Portal (Web, optional in MVP)**
  - Document verification queue, training content management, dispute handling, payouts.

### Data Flow (MVP)
1. Worker signs up → uploads documents → status `pending_verification`.
2. Admin verifies → worker completes training → status `active_available`.
3. Hirer posts request → system matches nearby workers → hirer calls or books.
4. Payment collected to app wallet → job completion triggers split payout.

### Scalability Notes
- Firestore + Cloud Functions good for MVP.
- Migrate to **Node.js + PostgreSQL** for complex analytics and billing.
- Use **GeoHash** indexing for location queries and caching for high availability.

---

## 2. Database Schema (Firestore)

### Collections

#### `users`
- `uid` (string)
- `role` (worker|hirer|admin)
- `name`
- `phone`
- `age` (worker only)
- `city`
- `createdAt`

#### `workers`
- `workerId` (uid)
- `status` (pending_verification | training_pending | active_available | active_booked | inactive)
- `categories` (array: helper, cleaner, loader, etc.)
- `aadhaarNumber`
- `aadhaarDocUrl`
- `photoUrl`
- `bankAccount` (masked)
- `ifsc`
- `rating` (float)
- `jobsCompleted` (int)
- `location` { `lat`, `lng`, `geoHash` }
- `lastActiveAt`

#### `hirers`
- `hirerId` (uid)
- `businessName`
- `address`
- `city`
- `walletBalance`

#### `jobs`
- `jobId`
- `hirerId`
- `workerId`
- `category`
- `status` (requested | booked | in_progress | completed | cancelled)
- `scheduledTime`
- `location` { `lat`, `lng` }
- `payment` { `total`, `commission`, `workerPayout`, `status` }
- `createdAt`

#### `trainingModules`
- `moduleId`
- `category`
- `videoUrl`
- `quiz` [{ `question`, `options`, `answerIndex` }]

#### `walletTransactions`
- `txnId`
- `userId`
- `type` (credit|debit)
- `amount`
- `reference` (jobId)
- `createdAt`

---

## 3. API List (Cloud Functions / Node.js)

### Auth & User
- `POST /auth/otp/request`
- `POST /auth/otp/verify`
- `GET /users/{uid}`

### Worker
- `POST /worker/onboard`
- `POST /worker/upload-docs`
- `POST /worker/complete-training`
- `POST /worker/set-availability`
- `GET /worker/nearby?category=&lat=&lng=&radius=`

### Hirer
- `POST /hirer/create-job`
- `POST /hirer/book-worker`
- `POST /hirer/cancel-job`
- `POST /hirer/remove-worker`

### Payments
- `POST /wallet/topup` (Razorpay order create)
- `POST /payment/webhook` (Razorpay webhook)
- `POST /job/complete` (trigger payout split)

---

## 4. Flutter Screen List

### Worker
1. OTP Login
2. Profile Setup
3. Document Upload
4. Training Video + Quiz
5. Status (Available/Booked)
6. Job History

### Hirer
1. OTP Login
2. Category + Location Selection
3. Nearby Worker List
4. Worker Profile
5. Booking Confirmation
6. Active Job Tracking

### Admin (Web/Mobile optional)
1. Verification Dashboard
2. Training Management
3. Dispute Resolution
4. Payouts

---

## 5. Sample Flutter UI Code (Login + Worker List)

```dart
import 'package:flutter/material.dart';

class LoginScreen extends StatelessWidget {
  const LoginScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Login')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            const TextField(
              decoration: InputDecoration(labelText: 'Phone Number'),
              keyboardType: TextInputType.phone,
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () {},
              child: const Text('Send OTP'),
            )
          ],
        ),
      ),
    );
  }
}

class WorkerListScreen extends StatelessWidget {
  final List<Map<String, dynamic>> workers = const [
    {'name': 'Ravi', 'distance': '1.2 km', 'rating': 4.5},
    {'name': 'Asha', 'distance': '2.0 km', 'rating': 4.2},
    {'name': 'Imran', 'distance': '3.5 km', 'rating': 4.8},
  ];

  const WorkerListScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Nearby Workers')),
      body: ListView.builder(
        itemCount: workers.length,
        itemBuilder: (context, index) {
          final worker = workers[index];
          return Card(
            child: ListTile(
              title: Text(worker['name']),
              subtitle: Text('${worker['distance']} • Rating ${worker['rating']}'),
              trailing: ElevatedButton(
                onPressed: () {},
                child: const Text('Book'),
              ),
            ),
          );
        },
      ),
    );
  }
}
```

---

## 6. Backend Logic Pseudocode

```pseudo
function onboardWorker(request):
  create worker profile
  set status = pending_verification
  notify admin verification queue

function verifyWorker(workerId):
  set status = training_pending
  assign training modules based on category

function completeTraining(workerId):
  if quiz passed:
    set status = active_available

function findNearbyWorkers(category, location, radius):
  query workers where status = active_available
  filter by category
  sort by distance + rating
  return list

function bookWorker(hirerId, workerId):
  create job record with status = booked
  set worker status = active_booked
  notify worker

function completeJob(jobId):
  update job status = completed
  calculate commission + payout
  credit worker wallet
  credit app commission

function removeWorker(jobId):
  update job status = cancelled
  set worker status = active_available
  refund if applicable
```

---

### MVP Notes
- Focus on **OTP onboarding, verification, and instant search**.
- Add **auto-availability toggle** + **rating system** in phase 2.
- Keep payment flow simple with Razorpay webhook confirmations.
