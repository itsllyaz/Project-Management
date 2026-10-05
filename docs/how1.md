Yes. Based on this schema, you can create both accounts directly in PostgreSQL. **But don't put a plain-text password into `password`**—your app expects bcrypt.

 ### 1\. Generate bcrypt hashes

 From your backend directory, first check whether `bcrypt` is installed:

```
cd ../backend
npm list bcrypt
```

 If it is installed, generate hashes with:

```
node -e "const bcrypt=require('bcrypt'); bcrypt.hash('Admin123456', 10).then(console.log)"
```

 And for the super admin:

```
node -e "const bcrypt=require('bcrypt'); bcrypt.hash('SuperAdmin123456', 10).then(console.log)"
```

 You'll get two long strings beginning roughly like:

```
$2b$10$...
```

 Copy those hashes.

 ### 2\. Insert the SUPER\_ADMIN and ADMIN

 Your database from the previous message is `internship_project`, and your PostgreSQL user is `llyaz`, so you can run:

```
docker exec -it project-management-db-1 psql -U llyaz -d internship_project
```

 Then:

```
INSERT INTO "User"
(name, email, password, role, permissions, status)
VALUES
(
  'Super Admin',
  'superadmin@example.com',
  'PASTE_SUPER_ADMIN_BCRYPT_HASH_HERE',
  'SUPER_ADMIN',
  '["manage_users","view_dashboard"]',
  'active'
),
(
  'Admin',
  'admin@example.com',
  'PASTE_ADMIN_BCRYPT_HASH_HERE',
  'ADMIN',
  '["manage_users","view_dashboard"]',
  'active'
);
```

 Then verify:

```
SELECT id, name, email, role, status, permissions
FROM "User"
ORDER BY id;
```

 You should see something like:

```
id | name        | email                    | role
---+-------------+--------------------------+------------
2  | Super Admin | superadmin@example.com   | SUPER_ADMIN
3  | Admin       | admin@example.com        | ADMIN
```

 ### 3\. Log in

 Use:

```
Super Admin
Email:    superadmin@example.com
Password: SuperAdmin123456
```

 or:

```
Admin
Email:    admin@example.com
Password: Admin123456
```

 ### Important

 There is one thing I'd verify **before you run the INSERT**: your application's exact permission names and password hashing implementation. Your schema allows any string for `role`, so the database itself won't stop an invalid role, but your application may expect specific permissions.

 Also, if `admin@example.com` or `superadmin@example.com` already exists, the `INSERT` will fail because `email` is `@unique`.

 If you paste your **`auth.controller.js` and the part containing `comparePassword`/password hashing**, I can give you the exact bootstrap command for your project rather than making assumptions.
