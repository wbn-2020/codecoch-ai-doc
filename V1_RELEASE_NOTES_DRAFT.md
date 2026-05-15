# V1_RELEASE_NOTES_DRAFT.md — CodeCoachAI V1 Release Notes Draft

## Release status

Status: Draft  
Target release: V1.0  
Current state: V1 code is complete; documentation still needs synchronization.

## 1. V1 positioning

CodeCoachAI V1 is the core closed-loop version of the AI interview training platform.

V1 focuses on:
- Question practice
- Resume/project-based interview preparation
- AI mock interview flow
- AI scoring and follow-up
- Interview report generation
- Admin-side question/prompt/system management

## 2. Completed user-side capabilities

- Account registration and login
- User profile management
- Question list and question detail
- Answer submission
- Favorite questions
- Wrong record review
- Resume creation/editing
- Project experience management
- Interview creation
- Interview room interaction
- AI scoring and feedback
- AI follow-up question
- Interview completion
- Interview report viewing
- Interview history viewing

## 3. Completed admin-side capabilities

- Admin dashboard
- User management
- Role/permission-related management
- Question management
- Category management
- Tag management
- Group management
- Prompt template management
- AI call log viewing
- System configuration

## 4. Completed backend capabilities

- Gateway routing
- Authentication
- User service
- Question service
- Resume service
- Interview service
- AI service
- System/admin service
- MySQL table initialization
- Development test data
- Redis/token support
- Nacos registration
- API documentation support

## 5. V1 acceptance checklist

Mark each item after real verification:

- [ ] Backend `mvn clean compile` passed
- [ ] Backend `mvn clean package -DskipTests` passed
- [ ] Frontend `npm run build` passed
- [ ] MySQL init script executed successfully
- [ ] Nacos services registered correctly
- [ ] Gateway routes work
- [ ] User registration/login/logout works
- [ ] Normal user cannot access admin routes
- [ ] Normal user cannot access other users' data
- [ ] Admin can manage questions/categories/tags/groups
- [ ] User can create/edit resume and project experience
- [ ] User can create an interview
- [ ] User can answer interview questions
- [ ] AI scoring returns correctly
- [ ] AI follow-up question returns correctly
- [ ] Interview can be finished
- [ ] Interview report can be generated and viewed
- [ ] AI call logs are recorded
- [ ] No old API paths appear in browser Network
- [ ] V1 docs are synchronized with actual code

## 6. Known limitations

- V1 may still rely on mock AI mode for stable local demo unless real model configuration is enabled.
- Advanced resume parsing is not included.
- Voice interview is not included.
- Vector search/embedding is not included.
- Advanced learning plan generation is not included.
- High-concurrency production hardening is not the V1 focus.

## 7. Release decision

Do not tag `v1.0` until:
1. Docs are synchronized.
2. Acceptance checklist is completed.
3. Known V1 limitations are explicitly documented.
4. V2 scope is separated from V1.
