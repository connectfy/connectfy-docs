# Client

## Overview
`connectfy-client` is the React web application for Connectfy. It handles authentication flows, profile management, user discovery, friendship management, settings, and the messenger UI. It talks to the API Gateway over HTTP and stores the access token in local state/local storage while relying on refresh cookies for silent renewal.

## Folder Structure
```text
connectfy-client/
├── src/
│   ├── assets/
│   ├── common/
│   │   ├── api/
│   │   ├── constants/
│   │   ├── enums/
│   │   ├── helpers/
│   │   ├── interfaces/
│   │   └── utils/
│   ├── components/
│   ├── context/
│   ├── hooks/
│   ├── layouts/
│   ├── modules/
│   │   ├── auth/
│   │   ├── messenger/
│   │   ├── profile/
│   │   ├── settings/
│   │   ├── termsAndConditions/
│   │   └── users/
│   ├── routes/
│   └── store/
```

Main areas:
- `common/api/axios.ts`: shared HTTP client with token refresh queueing.
- `routes/router.tsx`: root route composition.
- `modules/*`: feature-oriented screens and feature APIs.
- `layouts/*`: shared page shells.

## Architecture & Flow
1. App bootstraps providers for Redux, user/theme/context state, Google OAuth, and routing.
2. `api` attaches `Authorization`, language, and `x-device-id` headers.
3. On `401`, the response interceptor calls `/auth/refresh`, updates the access token, and retries queued requests.
4. Route guards redirect based on auth state and startup page.
5. Feature modules call API Gateway endpoints grouped in `API_ENDPOINTS`.

## Entities / Models
There are no persistence entities in the client. Important runtime models are:

### AuthState
- `access_token: string | null` - current bearer token
- `authenticateToken: string | null` - temporary auth token for sensitive flows

### Route Model
- Route constants under `ROUTER` define all public app paths.

### API Endpoint Map
- `API_ENDPOINTS` centralizes backend routes for auth, account, relationship, and file operations.

## Core Features
- Login, signup, Google auth, forgot/reset password
- Profile view and profile editing
- User search, friend requests, friends list, external user profile view
- General, privacy, notification, account, and background settings
- Messenger shell and navigation
- Global modal system, skeleton loaders, and context menus

## APIs
The client consumes API Gateway endpoints. Main groups:
- Auth: `/auth/*`
- User account actions: `/user/*`
- Profile and settings: `/account/*`
- Friendship actions: `/relationship/friendship/*`
- File uploads: `/files/presigned-upload`

Main routes:
- `/auth`, `/auth/login`, `/auth/signup`
- `/auth/login/verify`, `/auth/signup/verify`
- `/auth/forgot-password`, `/auth/reset-password`
- `/messenger`
- `/profile`
- `/users`, `/users/search`, `/users/friends`, `/users/requests`, `/users/profile/:id`
- `/settings`, `/settings/general`, `/settings/account`, `/settings/privacy`, `/settings/notification`, `/settings/background`
- `/terms-and-conditions`

## Technologies Used
- React 19
- TypeScript
- Vite
- React Router
- Zustand
- Redux Toolkit
- Axios
- i18next
- Flowbite / Tailwind
- Google OAuth

## Notes
- Access tokens are stored client-side; refresh tokens stay in cookies.
- `RequireAuth`, `InsideProfile`, and `RedirectMain` control most navigation rules.
- The route tree is feature-based, so new screens should usually be added through a feature module plus `routes/router.tsx`.
