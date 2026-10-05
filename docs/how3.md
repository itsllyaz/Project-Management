Ah, now we found the important detail. Your project has **both** libraries installed, but different parts use different ones:

 - `auth.controller.js` → `bcryptjs`
- `user.controller.js` → `bcryptjs`
- `src/utils/hash.js` → `bcrypt`

 For your login, **`auth.controller.js` is what matters**, so generate the password using **bcryptjs**.

 ### 1\. Generate the SUPER\_ADMIN password hash

 From `backend`:

```
node -e "const bcrypt=require('bcryptjs'); bcrypt.hash('SuperAdmin123456', 10).then(h=>console.log(h))"
```

 It will output something like:

```
$2b$10$.....................................................
```

 Copy the entire hash.

 ### 2\. Update your SUPER\_ADMIN

 Open PostgreSQL:

```
docker exec -it project-management-db-1 psql -U llyaz -d internship_project
```

 Then:

```
UPDATE "User"
SET password = 'PASTE_THE_HASH_HERE'
WHERE email = 'superadmin@example.com';
```

 Verify:

```
SELECT id, name, email, role, status
FROM "User"
WHERE email = 'superadmin@example.com';
```

 ### 3\. Test the password/hash directly

 Before trying the website, you can confirm the hash:

```
node -e "const bcrypt=require('bcryptjs'); bcrypt.compare('SuperAdmin123456', 'PASTE_THE_HASH_HERE').then(console.log)"
```

 It should return:

```
true
```

 Then log in:

```
Email:    superadmin@example.com
Password: SuperAdmin123456
```

 ### 4\. Do the same for your ADMIN

 Generate another hash:

```
node -e "const bcrypt=require('bcryptjs'); bcrypt.hash('Admin123456', 10).then(h=>console.log(h))"
```

 Then:

```
UPDATE "User"
SET password = 'PASTE_ADMIN_HASH_HERE'
WHERE email = 'admin@example.com';
```

 Now you should have:

```
SUPER_ADMIN
superadmin@example.com
SuperAdmin123456

ADMIN
admin@example.com
Admin123456
```

 One more thing: **your project has a small inconsistency** because `hash.js` uses `bcrypt` while the controllers use `bcryptjs`. They are generally compatible with bcrypt hashes, but I'd recommend standardizing the project on one library later. For getting you logged in now, use `bcryptjs` because that's what your authentication controller imports.
