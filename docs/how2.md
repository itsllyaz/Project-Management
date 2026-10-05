That usually means the account exists, but **the password hash in the database doesn't match what your login code is checking**.

 Since your app previously said it uses `bcrypt`, let's diagnose it instead of changing random things.

 ### 1\. Check the stored user

 Run:

```
docker exec -it project-management-db-1 psql -U llyaz -d internship_project
```

 Then:

```
SELECT id, name, email, role, status, password
FROM "User"
WHERE email = 'superadmin@example.com';
```

 The `password` should look something like:

```
$2b$10$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

 **Do not paste the complete password hash here.**

 ### 2\. Check which bcrypt package your backend uses

 From the backend directory:

```
npm list bcrypt bcryptjs
```

 Send me the output.

 Also run:

```
grep -R "comparePassword\|bcrypt.compare\|bcrypt.hash" -n src
```

 If your project is using something like:

```
bcrypt.compare(password, user.password)
```

 then the hash must have been generated using a compatible bcrypt implementation.

 ### 3\. The easiest test

 If you know the password you intended to use, for example:

```
SuperAdmin123456
```

 run this **from the backend directory**:

```
node -e "const bcrypt=require('bcrypt'); bcrypt.compare('SuperAdmin123456', 'PASTE_THE_HASH_FROM_DB_HERE').then(console.log)"
```

 You want:

```
true
```

 If you get:

```
false
```

 then **the hash/password combination is definitely the problem**.

 If you get `true` but the application still says `Invalid email or password`, then the problem is elsewhere in the login flow—possibly the email lookup, account status, JWT/auth middleware, or another validation.

 ### Send me these two outputs

```
npm list bcrypt bcryptjs
```

 and

```
grep -R "comparePassword\|bcrypt.compare\|bcrypt.hash" -n src
```

 Then I can tell you exactly what is wrong and give you the correct command to fix the SUPER\_ADMIN login.
