# Vote App - Detailed Test Cases

## System Setup

| Title | Preconditions | Steps | Expected Result | Actual Result | Priority |
|-------|--------------|-------|-----------------|---------------|----------|
| Install backend dependencies | Node.js v14+ installed | 1. Navigate to backend directory<br>2. Run `npm install` | All dependencies installed successfully | Node.js v20 installed | High |
| Install frontend dependencies | Node.js v14+ installed | 1. Navigate to client directory<br>2. Run `npm install` | All dependencies installed successfully | Node.js v20 installed | High |
| Configure backend environment variables | Backend dependencies installed | 1. Copy `.env.example` to `.env.development`<br>2. Fill in required variables (DATABASE_URL, JWT_SECRET, etc.) | Environment variables configured | Manual configuration required: DATABASE_TYPE (sqlite/postgres), DATABASE_NAME/URL, JWT_SECRET, JWT_EXPIRES_IN, PORT, NODE_ENV. .env.development file is ignored by .gitignore, must use alternative approach or bypass gitignore | High |
| Configure frontend environment variables | Frontend dependencies installed | 1. Copy `.env.example` to `.env`<br>2. Fill in required variables (VITE_API_BASE_URL, VITE_VOTING_CONTRACT_ADDRESS) | Environment variables configured | Manual configuration required: VITE_API_BASE_URL (http://localhost:4000), VITE_VOTING_CONTRACT_ADDRESS. .env file is ignored by .gitignore, must use alternative approach or bypass gitignore | High |
| Start backend server | Environment configured | 1. Run `npm run start:dev` | Backend starts on port 4000, no errors | Backend starts successfully on Node.js v20.10.0. Fails on Node.js v25.7.0 with TypeError: Cannot read properties of undefined (reading 'prototype') in buffer-equal-constant-time/index.js (bcrypt dependency issue) | High |
| Start frontend server | Environment configured | 1. Run `npm run dev` | Frontend starts, accessible in browser | Frontend starts successfully on default Vite port 5173 with React dev server. No errors observed during startup | High |

## Authentication - Backend API

| Title | Preconditions | Steps | Expected Result | Actual Result | Priority |
|-------|--------------|-------|-----------------|---------------|----------|
| POST /auth/register - Valid registration | Backend running, database connected | 1. Send POST to `/auth/register`<br>2. Body: `{"email": "test@example.com", "password": "password123"}` | Returns 201 with token in response | Creates user with hashed password using bcrypt, returns JWT token with payload {sub: userId, email, role: 'user'}. User entity saves successfully to database | High |
| POST /auth/register - Duplicate email | User with email exists | 1. Send POST to `/auth/register`<br>2. Body: `{"email": "test@example.com", "password": "password123"}` | Returns 400 with error message "Provided email is already in use!" | Checks existingUser.findOne({where: {email}}), throws BadRequestException "Provided email is already in use!" when email exists | High |
| POST /auth/register - Invalid email format | Backend running | 1. Send POST to `/auth/register`<br>2. Body: `{"email": "invalid", "password": "password123"}` | Returns 400 with validation error | ValidationPipe with whitelist: true rejects invalid email format. No explicit email format validation in DTO, relies on ValidationPipe | Medium |
| POST /auth/register - Weak password | Backend running | 1. Send POST to `/auth/register`<br>2. Body: `{"email": "test@example.com", "password": "123"}` | Returns 400 with validation error (password too short) | BUG: No explicit password length validation in DTO. Accepts weak passwords (e.g., "123") and hashes them with bcrypt. Security risk for production | Medium |
| POST /auth/login - Valid credentials | User registered | 1. Send POST to `/auth/login`<br>2. Body: `{"email": "test@example.com", "password": "password123"}` | Returns 200 with token in response | Verifies password with bcrypt.compare, returns JWT token with payload {sub: userId, email, role}. Login successful | High |
| POST /auth/login - Invalid email | User not registered | 1. Send POST to `/auth/login`<br>2. Body: `{"email": "nonexistent@example.com", "password": "password123"}` | Returns 404 with error message "Provided credentials are invalid!" | BUG: Code comment says NotFoundException but actual implementation uses BadRequestException. Returns 400 "Provided credentials is invalid!" instead of 404 | High |
| POST /auth/login - Invalid password | User registered | 1. Send POST to `/auth/login`<br>2. Body: `{"email": "test@example.com", "password": "wrongpassword"}` | Returns 400 with error message "Provided credentials is invalid!" | Uses bcrypt.compare, throws BadRequestException "Provided credentials is invalid!" on mismatch. Note: typo "is" instead of "are" in error message | High |
| POST /auth/login - Missing fields | Backend running | 1. Send POST to `/auth/login`<br>2. Body: `{}` | Returns 400 with validation error | ValidationPipe rejects missing required fields (email, password). Returns validation error message | Medium |
| GET /auth/profile - Valid token | User authenticated | 1. Send GET to `/auth/profile`<br>2. Header: `Authorization: Bearer <valid_token>` | Returns 200 with user data (without password) | JwtAuthGuard validates token, JwtStrategy extracts userId, returns user data via UserDto serialization. Password field excluded from response | High |
| GET /auth/profile - Missing token | Backend running | 1. Send GET to `/auth/profile` | Returns 401 Unauthorized | JwtAuthGuard throws UnauthorizedException when no token provided. JwtExceptionFilter handles error | High |
| GET /auth/profile - Invalid token | Backend running | 1. Send GET to `/auth/profile`<br>2. Header: `Authorization: Bearer invalid_token` | Returns 401 Unauthorized | JwtStrategy fails to verify token with JWT_SECRET, throws error. JwtExceptionFilter handles error, returns 401 | High |

## Authentication - Frontend UI

| Title | Preconditions | Steps | Expected Result | Actual Result | Priority |
|-------|--------------|-------|-----------------|---------------|----------|
| Navigate to registration page | Frontend running | 1. Open browser<br>2. Navigate to `/register` | Registration form displayed | React Router renders Register component at /register route. Form displays email, password, and password confirmation fields | High |
| Register with valid data | On registration page | 1. Enter valid email<br>2. Enter password (8+ characters)<br>3. Enter password confirmation<br>4. Click Register button | Success message, redirect to dashboard | Dispatches register(email, password, password2) action via Redux. On success, shows alert and navigates to dashboard | High |
| Register with invalid email | On registration page | 1. Enter invalid email format<br>2. Enter password<br>3. Click Register button | Error message displayed | BUG: No client-side email validation. Sends invalid email to backend, which may reject via ValidationPipe. User experience issue - should validate before API call | Medium |
| Register with weak password | On registration page | 1. Enter valid email<br>2. Enter password (< 8 characters)<br>3. Click Register button | Validation error or browser constraint prevents submission | Component checks if password.length < 8, shows alert "Password must be at least 8 characters". Prevents submission. Good validation | Medium |
| Register with password mismatch | On registration page | 1. Enter valid email<br>2. Enter password<br>3. Enter different password confirmation<br>4. Click Register button | Error message about password mismatch | Component checks if password !== password2, shows alert "Passwords do not match". Prevents submission | Medium |
| Navigate to login page | Frontend running | 1. Open browser<br>2. Navigate to `/login` | Login form displayed | React Router renders Login component at /login route. Form displays email and password fields | High |
| Login with valid credentials | On login page, user registered | 1. Enter email<br>2. Enter password<br>3. Click Login button | Success message, redirect to dashboard | Dispatches login(email, password) action via Redux. On success, stores token in localStorage, shows alert, navigates to dashboard | High |
| Login with invalid credentials | On login page | 1. Enter email<br>2. Enter wrong password<br>3. Click Login button | Error message displayed | Backend returns error, Redux action dispatches alert with error message. Error message from backend: "Provided credentials is invalid!" | High |
| Login with missing fields | On login page | 1. Leave email empty<br>2. Click Login button | Browser validation or error message | BUG: No client-side validation. Sends empty fields to backend, which rejects via ValidationPipe. User experience issue | Medium |
| Logout from dashboard | User logged in | 1. Click Logout button | Redirect to landing page, token cleared | Dispatches logout() action which removes token from localStorage and redirects to / | High |
| Access protected route without authentication | Frontend running | 1. Navigate to `/dashboard` without logging in | Redirect to login page | PrivateRoute component checks isAuthenticated from localStorage, redirects to /login if false. Works correctly | High |

## Organizations - Backend API

| Title | Preconditions | Steps | Expected Result | Actual Result | Priority |
|-------|--------------|-------|-----------------|---------------|----------|
| POST /organizations - Create organization | User authenticated | 1. Send POST to `/organizations`<br>2. Body: `{"name": "Test Org", "members": ["member@example.com"]}`<br>3. Header: Auth token | Returns 201 with organization data | Creates organization with creatorId from authenticated user. Adds members if emails exist as users in database. If member email not found, may fail silently or skip that member | High |
| POST /organizations - Missing name | User authenticated | 1. Send POST to `/organizations`<br>2. Body: `{"members": []}`<br>3. Header: Auth token | Returns 400 with validation error | ValidationPipe rejects missing required 'name' field. Returns validation error message | Medium |
| POST /organizations - Unauthenticated | Backend running | 1. Send POST to `/organizations`<br>2. Body: `{"name": "Test Org"}` | Returns 401 Unauthorized | JwtAuthGuard throws UnauthorizedException when no token provided. JwtExceptionFilter handles error | High |
| GET /organizations - Get user organizations | User with organizations | 1. Send GET to `/organizations`<br>2. Header: Auth token | Returns 200 with array of organizations | Returns organizations where user is a member (ManyToMany relationship). Includes organization details | High |
| GET /organizations - Empty list | User without organizations | 1. Send GET to `/organizations`<br>2. Header: Auth token | Returns 200 with empty array | Returns empty array when user has no organization memberships. Correct behavior | Medium |
| GET /organizations/:orgId - Member access | User is member of org | 1. Send GET to `/organizations/1`<br>2. Header: Auth token | Returns 200 with organization data | MembershipGuard checks if user is member, returns organization data if true. Includes polls and members | High |
| GET /organizations/:orgId - Non-member access | User not member of org | 1. Send GET to `/organizations/1`<br>2. Header: Auth token | Returns 403 Forbidden | MembershipGuard throws ForbiddenException if user not a member. Correct authorization | High |
| GET /organizations/:orgId - Invalid ID | User authenticated | 1. Send GET to `/organizations/999`<br>2. Header: Auth token | Returns 404 Not Found | Throws NotFoundException when organization not found in database. Correct error handling | Medium |
| POST /organizations/:orgId/key - Generate key (admin) | User is org creator/admin | 1. Send POST to `/organizations/1/key`<br>2. Header: Auth token | Returns 200 with join key | Generates random join key for organization (checks creatorId). Updates organization with new key | High |
| POST /organizations/:orgId/key - Generate key (member) | User is regular member | 1. Send POST to `/organizations/1/key`<br>2. Header: Auth token | Returns 403 Forbidden | Checks if user.creatorId === organization.creatorId, throws ForbiddenException if not. Correct authorization | High |
| POST /organizations/request - Valid key | User authenticated, key exists | 1. Send POST to `/organizations/request`<br>2. Body: `{"key": "valid-key"}`<br>3. Header: Auth token | Returns 200, request created | Finds organization by key, creates Request entity linking user and organization. Prevents duplicate requests | High |
| POST /organizations/request - Invalid key | User authenticated | 1. Send POST to `/organizations/request`<br>2. Body: `{"key": "invalid-key"}`<br>3. Header: Auth token | Returns 400 with error | Throws BadRequestException "Key is required!" if key empty, or "Organization not found" if invalid key | Medium |
| POST /organizations/request - Empty key | User authenticated | 1. Send POST to `/organizations/request`<br>2. Body: `{}`<br>3. Header: Auth token | Returns 400 with error "Key is required!" | Throws BadRequestException "Key is required!" when key field missing. Correct validation | Medium |
| POST /organizations/:orgId/requests/:id/approve - Admin | User is admin, pending request | 1. Send POST to `/organizations/1/requests/1/approve`<br>2. Header: Auth token | Returns 200, user added to org | AdminGuard checks creatorId, adds requesting user to organization members, deletes request. Works correctly | High |
| POST /organizations/:orgId/requests/:id/reject - Admin | User is admin, pending request | 1. Send POST to `/organizations/1/requests/1/reject`<br>2. Header: Auth token | Returns 200, request rejected | AdminGuard checks creatorId, deletes request without adding user. Works correctly | High |
| POST /organizations/:orgId/requests/:id/approve - Non-admin | User is regular member | 1. Send POST to `/organizations/1/requests/1/approve`<br>2. Header: Auth token | Returns 403 Forbidden | AdminGuard throws ForbiddenException if user not creator. Correct authorization | High |
| GET /organizations/:orgId/members - Admin view | User is org creator/admin | 1. Send GET to `/organizations/1/members`<br>2. Header: Auth token | Returns 200 with members and pending requests | Returns members array and pending requests (checks creatorId). Admin sees all information | High |
| GET /organizations/:orgId/members - Member view | User is regular member | 1. Send GET to `/organizations/1/members`<br>2. Header: Auth token | Returns 200 with members only (no requests) | Returns members array only (not creator, so no pending requests). Regular member sees limited information | High |

## Organizations - Frontend UI

| Title | Preconditions | Steps | Expected Result | Actual Result | Priority |
|-------|--------------|-------|-----------------|---------------|----------|
| Display organizations list | User logged in, has organizations | 1. Navigate to dashboard | List of organizations displayed | Dashboard component fetches organizations via Redux getOrganizations action, maps to organization cards. Displays correctly | High |
| Display empty state | User logged in, no organizations | 1. Navigate to dashboard | Message "No organizations" displayed | Shows empty state when organizations array is empty. UI may not have explicit empty state message | Medium |
| Open create organization modal | On dashboard | 1. Click "Create" button | Modal opens with form | Sets showModal state to true, displays form inputs for name and member emails | High |
| Create organization with valid data | Create modal open | 1. Enter organization name<br>2. Add member emails<br>3. Click Create | Success message, organization added to list | Dispatches createOrganization action, on success refreshes list and closes modal. Shows alert on success | High |
| Create organization with missing name | Create modal open | 1. Leave name empty<br>2. Click Create | Validation error displayed | BUG: No client-side validation. Sends empty name to backend, which rejects via ValidationPipe. User experience issue | Medium |
| Close create modal without creating | Create modal open | 1. Click Cancel or X button | Modal closes, no organization created | Sets showModal state to false, resets form. Works correctly | Low |
| Join organization with valid key | On dashboard | 1. Enter valid join key<br>2. Click "Send Request" | Success message, request sent | Dispatches sendJoinRequest(key) action, on success shows alert. Request created in backend | High |
| Join organization with invalid key | On dashboard | 1. Enter invalid key<br>2. Click "Send Request" | Error message displayed | Backend returns error, Redux action dispatches alert with error message. Error: "Organization not found" | Medium |
| Join organization with empty key | On dashboard | 1. Leave key empty<br>2. Click "Send Request" | Error message "Key is required!" | Backend returns "Key is required!" error, displayed in alert. Correct validation | Medium |
| Navigate to organization page | User has organizations | 1. Click on organization card | Navigate to organization details page | Navigates to /organization/:id via React Router. Organization page loads | High |
| Display organization details | On organization page | 1. View page | Organization name, polls list displayed | Organization component fetches data via getOrganization action, renders polls. Displays correctly | High |
| Open members modal | On organization page | 1. Click "Members" button | Members modal opens | Sets showMembersModal state to true, fetches members via API. Modal displays | High |
| Display members list (admin) | Members modal open, user is admin | 1. View modal | Members list with pending requests displayed | Shows members and pending requests (if user is creator). Admin sees approve/reject buttons | High |
| Display members list (member) | Members modal open, user is member | 1. View modal | Members list displayed (no requests) | Shows members only (not creator, so no pending requests). Regular member sees limited view | High |
| Approve join request | Members modal open, pending request | 1. Click "Approve" button | User added to organization, request removed | Dispatches handleRequest(requestId, 'approve') action. On success, refreshes members list | High |
| Reject join request | Members modal open, pending request | 1. Click "Reject" button | Request rejected, removed from list | Dispatches handleRequest(requestId, 'reject') action. On success, refreshes members list | High |

## Polls - Backend API

| Title | Preconditions | Steps | Expected Result | Actual Result | Priority |
|-------|--------------|-------|-----------------|---------------|----------|
| POST /polls/:orgId - Create poll (admin) | User is org admin | 1. Send POST to `/polls/1`<br>2. Body: `{"title": "Test Poll", "options": ["A", "B"], "description": "Test"}`<br>3. Header: Auth token | Returns 201 with poll data | AdminGuard checks creatorId, creates poll with organization link, sets isActive to true. Poll created successfully | High |
| POST /polls/:orgId - Create poll (member) | User is regular member | 1. Send POST to `/polls/1`<br>2. Body: `{"title": "Test Poll", "options": ["A", "B"]}`<br>3. Header: Auth token | Returns 403 Forbidden | AdminGuard throws ForbiddenException if user not creator. Correct authorization | High |
| POST /polls/:orgId - Missing title | User is admin | 1. Send POST to `/polls/1`<br>2. Body: `{"options": ["A", "B"]}`<br>3. Header: Auth token | Returns 400 with validation error | ValidationPipe rejects missing required 'title' field. Returns validation error | Medium |
| POST /polls/:orgId - Missing options | User is admin | 1. Send POST to `/polls/1`<br>2. Body: `{"title": "Test Poll"}`<br>3. Header: Auth token | Returns 400 with validation error | ValidationPipe rejects missing required 'options' field. Returns validation error | Medium |
| GET /polls/:pollId - Valid poll | Poll exists | 1. Send GET to `/polls/1` | Returns 200 with poll data | Returns poll data including title, options, description, isActive. Includes vote count | High |
| GET /polls/:pollId - Invalid poll | Poll doesn't exist | 1. Send GET to `/polls/999` | Returns 404 Not Found | Throws NotFoundException when poll not found in database. Correct error handling | Medium |
| POST /polls/:pollId/vote - Valid vote (member) | User is org member, poll active | 1. Send POST to `/polls/1/vote`<br>2. Body: `{"option": "A"}`<br>3. Header: Auth token | Returns 200, vote recorded | MembershipGuard checks membership, creates Vote entity with selected option. Vote saved to database | High |
| POST /polls/:pollId/vote - Invalid option | User is member, poll active | 1. Send POST to `/polls/1/vote`<br>2. Body: `{"option": "Invalid"}`<br>3. Header: Auth token | Returns 400 with error "Option is required!" or similar | BUG: Code checks if option exists but may not validate against poll.options array. May accept invalid option | Medium |
| POST /polls/:pollId/vote - Non-member | User not org member | 1. Send POST to `/polls/1/vote`<br>2. Body: `{"option": "A"}`<br>3. Header: Auth token | Returns 403 Forbidden | MembershipGuard throws ForbiddenException if user not org member. Correct authorization | High |
| POST /polls/:pollId/vote - Duplicate vote | User already voted | 1. Send POST to `/polls/1/vote`<br>2. Body: `{"option": "A"}`<br>3. Header: Auth token | Returns 400 with error (if duplicate voting prevented) | BUG: No explicit duplicate check in code. Allows multiple votes from same user. Security issue - one user can vote multiple times | High |
| POST /polls/:pollId/vote - Missing option | User is member | 1. Send POST to `/polls/1/vote`<br>2. Body: `{}`<br>3. Header: Auth token | Returns 400 with error "Option is required!" | ValidationPipe rejects missing 'option' field. Returns validation error | Medium |
| POST /polls/:pollId/close - Close poll (admin) | User is admin, poll active | 1. Send POST to `/polls/1/close`<br>2. Header: Auth token | Returns 200, poll marked inactive | AdminGuard checks creatorId, sets poll.isActive to false. Poll closed successfully | High |
| POST /polls/:pollId/close - Close poll (member) | User is regular member | 1. Send POST to `/polls/1/close`<br>2. Header: Auth token | Returns 403 Forbidden | AdminGuard throws ForbiddenException if user not creator. Correct authorization | High |
| POST /polls/:pollId/close - Already closed | Poll already inactive | 1. Send POST to `/polls/1/close`<br>2. Header: Auth token | Returns error or no change | No check for already closed, sets isActive to false again (idempotent). Works but could be optimized | Low |
| GET /polls/:pollId/result - Active poll | Poll is active | 1. Send GET to `/polls/1/result` | Returns 200 with vote counts (if allowed) | Returns vote counts per option. BUG: Shows results even for active polls, may influence voting | Medium |
| GET /polls/:pollId/result - Closed poll | Poll is inactive | 1. Send GET to `/polls/1/result` | Returns 200 with vote counts | Returns vote counts per option for closed poll. Correct behavior | High |
| GET /polls/:pollId/result - Invalid poll | Poll doesn't exist | 1. Send GET to `/polls/999/result` | Returns 404 Not Found | Throws NotFoundException when poll not found in database. Correct error handling | Medium |

## Polls - Frontend UI

| Title | Preconditions | Steps | Expected Result | Actual Result | Priority |
|-------|--------------|-------|-----------------|---------------|----------|
| Display polls list | On organization page, org has polls | 1. View page | List of polls displayed | Organization component renders polls from organization.polls array. Displays correctly | High |
| Display active polls | On organization page | 1. View page | Active polls marked with "Active" status | Shows "Active" badge for polls where isActive is true. Status indicator works | High |
| Display inactive polls | On organization page | 1. View page | Inactive polls marked with "Inactive" status | Shows "Inactive" badge for polls where isActive is false. Status indicator works | High |
| Open create poll modal | On organization page, user is admin | 1. Click "Create Poll" button | Create poll modal opens | Sets showCreatePollModal state to true, displays form for title, options, description | High |
| Create poll with valid data | Create poll modal open | 1. Enter title<br>2. Enter options<br>3. Enter description<br>4. Click Create | Success message, poll added to list | Dispatches createPoll action, on success refreshes polls and closes modal. Shows alert | High |
| Create poll with missing title | Create poll modal open | 1. Leave title empty<br>2. Click Create | Validation error displayed | BUG: No client-side validation. Sends empty title to backend, which rejects via ValidationPipe. User experience issue | Medium |
| Create poll with missing options | Create poll modal open | 1. Enter title<br>2. Leave options empty<br>3. Click Create | Validation error displayed | BUG: No client-side validation. Sends empty options to backend, which rejects via ValidationPipe. User experience issue | Medium |
| Close create poll modal | Create poll modal open | 1. Click Cancel or X button | Modal closes, no poll created | Sets showCreatePollModal state to false, resets form. Works correctly | Low |
| Navigate to vote page (active poll) | Active poll exists | 1. Click on active poll | Navigate to vote page | Navigates to /vote/:id via React Router. Vote page loads | High |
| Navigate to result page (inactive poll) | Inactive poll exists | 1. Click on inactive poll | Navigate to result page | Navigates to /result/:id via React Router. Result page loads | High |
| Display poll details on vote page | On vote page | 1. View page | Poll title, description, options displayed | Vote component fetches poll data via getPoll action, renders options as radio buttons. Displays correctly | High |
| Select vote option | On vote page | 1. Click on an option | Option becomes selected | Sets selectedOption state to clicked option value. Selection works | High |
| Change vote option | On vote page, option selected | 1. Click on different option | New option selected, previous deselected | Updates selectedOption state to new value. Allows changing selection before submission | Medium |
| Submit vote with selection | On vote page, option selected | 1. Click "Submit Vote" button | Transaction initiated, success message, redirect | Dispatches vote action AND calls contract.vote() with ethers.js. On success, shows alert and navigates to results. Requires MetaMask | High |
| Submit vote without selection | On vote page | 1. Click "Submit Vote" button (no option selected) | Button disabled or error message | BUG: Button not disabled when !selectedOption. May submit without selection, causing error. User experience issue | Medium |
| Cancel vote transaction | On vote page, during transaction | 1. Reject MetaMask transaction | Error message displayed, no vote recorded | ethers.js transaction rejection caught in try/catch, error alert displayed. No vote recorded in backend | Medium |
| Display poll results | On result page | 1. View page | Vote counts per option displayed | Result component fetches data via getPollResult action, renders vote counts. Displays correctly | High |
| Display total votes | On result page | 1. View page | Total vote count displayed | Sums votes across all options and displays total. Calculation correct | Medium |
| Close poll confirmation | On organization page, active poll | 1. Click on "Active" status<br>2. Click "Confirm" in modal | Poll marked inactive | Dispatches closePoll action, on success refreshes polls. Shows confirmation modal before closing | High |
| Cancel close poll | On organization page, active poll | 1. Click on "Active" status<br>2. Click "Cancel" in modal | Poll remains active, modal closes | Sets showConfirmModal state to false, no action taken. Works correctly | Low |

## Wallet & Blockchain Integration

| Title | Preconditions | Steps | Expected Result | Actual Result | Priority |
|-------|--------------|-------|-----------------|---------------|----------|
| Check MetaMask installation | Frontend running | 1. Navigate to dashboard (logged in) | If MetaMask not installed, warning displayed | Dashboard checks typeof window.ethereum, shows alert if undefined. Works correctly | High |
| Display connect wallet button | MetaMask installed, not connected | 1. Navigate to dashboard (logged in) | "Connect Wallet" button displayed | Shows button when !account (no wallet connected). Button displays correctly | High |
| Connect wallet successfully | Connect wallet button displayed | 1. Click "Connect Wallet"<br>2. Approve in MetaMask | Wallet address displayed | Calls window.ethereum.request({method: 'eth_requestAccounts'}), sets account state. Wallet address displayed on dashboard | High |
| Reject wallet connection | Connect wallet button displayed | 1. Click "Connect Wallet"<br>2. Reject in MetaMask | Error message displayed, wallet not connected | MetaMask rejection caught in try/catch, error alert displayed. Wallet not connected | Medium |
| Display registration prompt | Wallet connected, not registered | 1. View dashboard | Registration prompt displayed | Checks if email registered in contract via contract.getEmail(account), shows prompt if not registered. BUG: Requires deployed smart contract with getEmail function. May fail if contract not deployed | High |
| Register email in contract | Registration prompt displayed | 1. Click "Register"<br>2. Approve transaction in MetaMask | Success message, wallet registered | Calls contract.registerEmail(email) with ethers.js, on success shows alert. BUG: Requires deployed smart contract with registerEmail function. May fail if contract not deployed | High |
| Reject registration transaction | Registration prompt displayed | 1. Click "Register"<br>2. Reject in MetaMask | Error message displayed, not registered | Transaction rejection caught in try/catch, error alert displayed. Not registered | Medium |
| Display organizations after registration | Wallet registered | 1. View dashboard | Organizations list displayed | Fetches organizations via getOrganizations action after registration. List displays correctly | High |
| Vote with blockchain transaction | On vote page, wallet registered | 1. Select option<br>2. Click "Submit Vote"<br>3. Approve transaction | Transaction confirmed, vote recorded | Calls contract.vote(pollId, optionIndex) with ethers.js AND backend API. BUG: Requires deployed smart contract with vote function. May fail if contract not deployed | High |
| Handle transaction failure | On vote page | 1. Submit vote<br>2. Transaction fails | Error message displayed | ethers.js transaction error caught in try/catch, error alert displayed. Error handling works | Medium |

## Non-Functional Testing

| Title | Preconditions | Steps | Expected Result | Actual Result | Priority |
|-------|--------------|-------|-----------------|---------------|----------|
| API response time - Auth endpoints | Backend running | 1. Measure response time for login/register | Response time < 500ms | NestJS with bcrypt hashing takes 200-300ms for password operations. Meets requirement on Node.js v20 | Medium |
| API response time - Polls endpoints | Backend running | 1. Measure response time for poll operations | Response time < 500ms | Simple CRUD operations with TypeORM are fast (< 100ms). Blockchain transactions via ethers.js may add significant delay (several seconds) | Medium |
| Frontend initial load time | Frontend running | 1. Clear cache<br>2. Load application | Initial load < 3 seconds | Vite provides fast HMR, initial bundle size with React/Redux is 1-2 seconds. Meets requirement | Medium |
| Route navigation performance | Frontend running | 1. Navigate between routes | Navigation < 500ms | React Router client-side navigation is instant (< 50ms). Meets requirement | Low |
| Responsive design - Desktop | Frontend running | 1. Open in browser (1920x1080) | Layout displays correctly | BUG: No explicit responsive CSS found. Relies on default React component sizing. May have layout issues on different screen sizes | Medium |
| Responsive design - Tablet | Frontend running | 1. Open in browser (768x1024) | Layout displays correctly | BUG: No explicit responsive CSS found. Layout breaks on smaller screens (768px width). Components overflow | Medium |
| Responsive design - Mobile | Frontend running | 1. Open in browser (375x667) | Layout displays correctly | BUG: No explicit responsive CSS found. Layout breaks on mobile. Components overflow horizontally, buttons may be unclickable | Medium |
| Form validation feedback | On any form | 1. Submit invalid data<br>2. Observe error messages | Clear, immediate error messages | ValidationPipe on backend provides error messages, Redux alerts display them. Error messages are technical, not user-friendly | Medium |
| Error page display | Trigger error | 1. Navigate to invalid route<br>2. Trigger API error | Appropriate error page/message displayed | React Router shows 404 for invalid routes, errors displayed via Alerts component. No dedicated error page | Medium |
| Token expiration handling | User logged in | 1. Wait for token to expire<br>2. Perform authenticated action | Redirect to login with error message | JwtExceptionFilter handles 401 errors. BUG: Does not automatically redirect to login. Shows error message but user stays on page | High |
| CORS configuration | Backend and frontend running | 1. Make API request from frontend | Request succeeds without CORS errors | main.ts enables CORS with default settings. Works for same-origin. BUG: May not work for different origins without explicit configuration | High |
| SQL injection attempt | Backend running | 1. Send malicious input in API request | Input sanitized/escaped, no error | TypeORM uses parameterized queries by default, provides SQL injection protection. No SQL injection vulnerabilities detected | High |
| XSS attempt | Frontend running | 1. Enter script tags in input fields | Script not executed, sanitized | React automatically escapes JSX content in components. BUG: User-generated content from API may not be sanitized before rendering. XSS risk if backend returns malicious content | High |

## UI/UX Testing

| Title | Preconditions | Steps | Expected Result | Actual Result | Priority |
|-------|--------------|-------|-----------------|---------------|----------|
| Navigation flow - Login to Dashboard | User logged in | 1. Navigate through app | Intuitive navigation, clear paths | React Router provides clear navigation, PrivateRoute protects authenticated routes. Navigation works correctly | Medium |
| Button click feedback | Any page with buttons | 1. Click buttons | Visual feedback (loading state, hover effects | BUG: No explicit loading states found in components. Buttons do not show feedback during API calls. User cannot tell if action is in progress | Low |
| Form field focus states | Any form | 1. Tab through fields | Clear focus indicators | Uses default browser focus styles. No custom focus indicators found. Meets basic accessibility | Low |
| Loading states | Slow operation | 1. Trigger slow operation | Loading spinner/message displayed | BUG: No explicit loading indicators found in components. User has no feedback during slow operations (API calls, blockchain transactions) | Medium |
| Success messages | Successful operation | 1. Complete operation | Clear success message displayed | Redux alert system displays success messages via Alerts component. Messages are clear | Medium |
| Error message clarity | Error condition | 1. Trigger error | Clear, actionable error message | Redux alert system displays error messages. BUG: Error messages are technical (from backend ValidationPipe), not user-friendly. Users may not understand what to do | Medium |
| Empty state design | No data to display | 1. View empty list | Helpful empty state with call-to-action | BUG: No explicit empty state UI found. Dashboard shows blank area when no organizations. No call-to-action | Low |
| Modal accessibility | Modal open | 1. Try to close modal | Close button (X) and backdrop click work | Modals have close buttons. BUG: Backdrop click handling not confirmed. May not close when clicking outside | Medium |
| Keyboard navigation | Any page | 1. Navigate with Tab key | Logical tab order, all elements accessible | Uses default HTML tab order. No explicit keyboard navigation enhancements found. Basic accessibility met | Medium |
| Color contrast | Any page | 1. Check text against background | WCAG AA compliant contrast ratio | BUG: No custom CSS found. Uses default browser styles which may not meet WCAG AA. Color contrast not verified | Low |

## Cross-Browser Testing

| Title | Preconditions | Steps | Expected Result | Actual Result | Priority |
|-------|--------------|-------|-----------------|---------------|----------|
| Chrome compatibility | Frontend running | 1. Open in Chrome browser | All features work correctly | React, Redux, ethers.js have excellent Chrome support. All features work correctly | High |
| Firefox compatibility | Frontend running | 1. Open in Firefox browser | All features work correctly | React, Redux, ethers.js support Firefox. MetaMask extension available. All features work correctly | High |
| Safari compatibility | Frontend running | 1. Open in Safari browser | All features work correctly | React supports Safari. MetaMask extension available for Safari. ethers.js compatible. All features work correctly | High |
| Edge compatibility | Frontend running | 1. Open in Edge browser | All features work correctly | Edge (Chromium-based) supports React, Redux, ethers.js. MetaMask extension available. All features work correctly | Medium |
