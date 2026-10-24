# Thalia — Afro Hair Care

Thalia is a full-stack web application designed to help users build, organize, and maintain personalized Afro hair-care routines.

The application combines a responsive hair-care calendar and routine manager with secure user authentication and cloud-based data storage using a serverless AWS architecture.

Thalia is currently in beta testing.

---

## Overview

Hair-care routines can involve multiple wash days, treatments, moisturizing sessions, protective styles, products, and personal hair goals.

Thalia provides one place for users to organize these activities and build a routine that works for them.

Users can:

- Create personalized hair-care practices
- Schedule one-time, weekly, and monthly routines
- Track completed hair-care activities
- Manage products used in their routines
- Define hair-care goals
- Store hair characteristics such as texture, density, and porosity
- Access their saved routine after signing back in
- Use the application across desktop and mobile devices

---

## Live Application

## User Testing & Production Validation.

https://dreamy-bublanina-6d0d6b.netlify.app/

> Thalia is currently undergoing controlled beta testing. Features and interface elements may change as feedback is collected.

---

## Architecture

Thalia uses a serverless AWS backend with a static JavaScript frontend.

```text
                     ┌─────────────────────┐
                     │       User          │
                     │ Desktop / Mobile    │
                     └──────────┬──────────┘
                                │
                                ▼
                     ┌─────────────────────┐
                     │   Thalia Web App    │
                     │ HTML / CSS / JS     │
                     │      Netlify        │
                     └───────┬──────┬──────┘
                             │      │
                  Sign in    │      │ API requests
                             │      │
                             ▼      ▼
                  ┌─────────────┐  ┌─────────────────┐
                  │ AWS Cognito │  │ API Gateway     │
                  │             │  │ GET /data       │
                  │ User Auth   │  │ PUT /data       │
                  └─────────────┘  └────────┬────────┘
                                           │
                                      JWT validation
                                           │
                                           ▼
                                  ┌──────────────────┐
                                  │   AWS Lambda     │
                                  │   Python API     │
                                  └────────┬─────────┘
                                           │
                                           ▼
                                  ┌──────────────────┐
                                  │ Amazon DynamoDB  │
                                  │ Per-user data    │
                                  └──────────────────┘
```

---

## Tech Stack

### Frontend

- HTML5
- CSS3
- Vanilla JavaScript
- Responsive CSS
- Browser Local Storage
- Session Storage

### AWS Backend

- Amazon Cognito
- Amazon API Gateway
- AWS Lambda
- Amazon DynamoDB
- Amazon CloudWatch
- AWS IAM

### Deployment

- Netlify
- GitHub

---

## Authentication

Thalia uses **Amazon Cognito** for user authentication.

The frontend implements the OAuth 2.0 **Authorization Code Grant with PKCE**.

PKCE allows the browser application to authenticate users without embedding a client secret in frontend code.

The authentication flow is approximately:

```text
User
  │
  ▼
Thalia
  │
  │ Generate PKCE verifier + challenge
  ▼
Amazon Cognito
  │
  │ User authentication
  ▼
Authorization Code
  │
  ▼
Thalia
  │
  │ Exchange code + PKCE verifier
  ▼
Access Token
  │
  ▼
API Gateway
```

Authentication state is maintained using browser session storage.

Expired sessions are detected and cleared, requiring the user to authenticate again.

---

## API Security

Thalia's API routes are protected using an API Gateway JWT authorizer.

The frontend sends the user's Cognito access token with authenticated requests:

```http
Authorization: Bearer <access-token>
```

API Gateway validates the token before allowing the request to reach the Lambda function.

The application currently exposes two authenticated operations:

```text
GET /data
PUT /data
```

This means database access is not exposed directly to the browser.

---

## Per-User Data Isolation

A major design requirement for Thalia was ensuring that authenticated users could only retrieve their own application data.

After API Gateway validates the JWT, the Lambda function extracts the authenticated Cognito user's `sub` claim.

That identifier is used as the DynamoDB partition key.

Conceptually:

```text
Authenticated User
       │
       ▼
 Cognito JWT
       │
       ▼
API Gateway
       │
       ▼
Lambda
       │
       │ extract user "sub"
       ▼
DynamoDB
       │
       ▼
Only that user's records
```

The client does not choose which user's partition to query.

This design was tested using multiple user accounts to confirm that data belonging to one account was not returned to another.

---

## DynamoDB Data Model

Thalia stores cloud-managed application data in the `ThaliaUserData` DynamoDB table.

The table uses:

```text
Partition key: userId
Sort key:      recordedId
```

Records include:

```text
rituals
done
products
hairGoals
hairPreferences
```

This allows multiple categories of application data to be stored under the authenticated user's partition.

---

## Local and Cloud Data

Thalia uses a hybrid local/cloud storage model.

### Cloud-managed data

The following data is synchronized with the authenticated user's account:

- Hair-care practices
- Practice completion history
- Products
- Hair goals
- Hair texture
- Hair density
- Hair porosity

### Device-local data

Some profile information currently remains on the user's device.

This includes information such as:

- Profile name
- Age
- Location
- Email
- Gender information
- Profile photo

The distinction between local and cloud data is intentional and is being evaluated during the beta period.

---

## User Session Protection

During development, an important privacy issue was identified.

Cloud-restored data was being cached locally in the browser. Although cloud accounts were correctly isolated, signing out could leave the previous user's cached routine visible on a shared device.

The logout process was redesigned to selectively remove account-specific cached data.

Instead of clearing all browser storage, Thalia removes only cloud-managed user data while preserving intentionally device-local profile information.

The same protection is applied when a user's authentication session expires.

---

## Features

### Hair-Care Calendar

Users can view their scheduled hair-care practices through a calendar interface.

Practices can be scheduled:

- Once on a specific date
- Weekly
- Monthly

### Practice Tracking

Users can mark scheduled practices as completed.

Completion history contributes to progress and care-streak information.

### Routine Management

Users can build and modify their weekly hair-care routine.

Practices can represent activities such as:

- Wash days
- Deep treatments
- Everyday care
- Midweek resets
- Protective styles

### Product Management

Users can maintain a list of products currently used in their routine.

Products can also be associated with individual hair-care practices.

### Hair Goals

Users can track goals including:

- Length retention and growth
- Volume and texture maintenance
- Hydration
- Curl definition and moisture

### Hair Profile

Users can record:

- Hair texture
- Hair density
- Hair porosity

The profile interface also supports additional device-local profile information and a profile image.

---

## Responsive Design

Thalia was designed and manually tested across desktop and mobile layouts.

Mobile testing covered:

- Today dashboard
- Calendar
- My Routine
- Practice creation modal
- Profile form
- Product management
- Hair goals
- Authentication
- Cloud save and restore
- Practice deletion

A final desktop regression test was also performed after mobile testing.

---

## Cloud Synchronization

When an authenticated user makes a cloud-managed change, Thalia saves the change locally and synchronizes it through the API.

```text
User action
    │
    ▼
Local application state
    │
    ▼
Local Storage
    │
    ▼
API Gateway
    │
    ▼
Lambda
    │
    ▼
DynamoDB
```

When the user signs in again, Thalia retrieves the account's records from DynamoDB and rebuilds the application's local state.

Cloud restoration has been tested across sign-out/sign-in and browser refresh cycles.

---

## AWS Security Measures

Security was reviewed before beginning beta testing.

Implemented measures include:

- Cognito-based authentication
- Authorization Code + PKCE
- JWT-protected API routes
- Per-user DynamoDB partitioning
- No Cognito client secret in the frontend
- No AWS access keys in frontend code
- Scoped Lambda IAM permissions
- Session-expiration handling
- Logout cache isolation
- DynamoDB deletion protection
- DynamoDB Point-in-Time Recovery
- CloudWatch log retention configuration
- Restricted API CORS origins

---

## Testing

Thalia has undergone functional and security-oriented testing covering:

- Account registration
- Login
- Logout
- Session expiration
- API authorization
- Cloud data save
- Cloud data restoration
- Multiple-user data isolation
- Routine creation
- Routine deletion
- Product synchronization
- Hair-goal synchronization
- Hair-preference synchronization
- Mobile responsiveness
- Desktop regression testing

A temporary practice was also created through the mobile interface, synchronized to the cloud, restored after refresh, deleted, and confirmed absent after another refresh.

---

## Beta Testing

Thalia has entered a controlled beta-testing stage.

The initial beta focuses on:

- First-time usability
- Account creation
- Routine creation
- Calendar usability
- Product management
- Hair goals
- Returning-user experience
- Mobile usability
- Data persistence
- Unexpected errors or confusing workflows

Feedback from beta testers will be used to prioritize improvements before wider release.

---

## Engineering Challenges

### Designing Secure User Isolation

One of the most important backend challenges was ensuring that the frontend could never request another user's data simply by supplying another user ID.

The solution was to derive the database partition key from the authenticated Cognito JWT inside the backend rather than trusting an identifier supplied by the browser.

### Local vs Cloud State

The original application stored its state in `localStorage`.

Moving to authenticated cloud storage required introducing synchronization without replacing the existing application.

The migration was implemented incrementally so the working frontend could be preserved while AWS services were introduced.

### DynamoDB Integration

Integrating the API with DynamoDB required coordinating the application's record model with the table's partition and sort-key structure.

Debugging this integration reinforced the importance of keeping infrastructure schemas and application code aligned.

### Session and Shared-Device Privacy

Cloud isolation alone was not sufficient.

Testing showed that browser caching could expose the previous user's cloud-restored information after logout on a shared browser.

Thalia therefore implements selective cache cleanup during logout and session expiration.

---

## Reliability and Data Protection

The DynamoDB table has:

- Deletion protection enabled
- Point-in-Time Recovery enabled

CloudWatch is used for Lambda logging and troubleshooting.

These protections were added as part of the application's production-readiness review.

---

## Current Status

**Beta**

The core application is operational and deployed.

Completed milestones include:

- Responsive frontend
- AWS authentication
- Serverless API
- Cloud database
- Per-user data isolation
- Session management
- Production deployment
- Security review
- Mobile usability testing
- Branding integration
- Controlled beta preparation

Further launch preparation and improvements are being informed by beta feedback.

---

## Future Improvements

Potential post-beta improvements include:

- Improved cloud-sync failure feedback
- Custom production domain
- Expanded accessibility testing
- Additional routine personalization
- Improved onboarding
- Notification/reminder capabilities
- Additional account-security options
- Privacy and account-management improvements
- Progressive Web App capabilities

Feature priorities will be influenced by beta feedback rather than added solely for technical complexity.

---

## What I Learned

Building Thalia provided practical experience across the complete lifecycle of a cloud application:

- Designing a product around a real user problem
- Building a responsive frontend without a JavaScript framework
- Designing serverless AWS architecture
- Implementing OAuth authentication with PKCE
- Protecting APIs with JWT authorization
- Designing DynamoDB data models
- Implementing per-user data isolation
- Managing local and cloud application state
- Debugging distributed frontend/API/database issues
- Applying least-privilege IAM concepts
- Testing authentication and session edge cases
- Preparing an application for real beta users
- Deploying and maintaining a production web application

The project evolved from a browser-based prototype into an authenticated, cloud-backed application while preserving the existing user experience.

---

## Author

Built as a full-stack cloud portfolio project demonstrating frontend development, serverless AWS architecture, authentication, database design, security, deployment, and product testing.
