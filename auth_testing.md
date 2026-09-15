# Auth-Gated Testing Playbook — The Gali Mohalla

Google OAuth (Emergent-managed). App users have a custom `user_id`; admin actions require `is_admin: true`.
IMPORTANT: The site uses a "first user to log in becomes admin" rule. For testing, insert a user with `is_admin: true` directly.

## Create Test Admin User + Session (pymongo)
```bash
cd /app/backend && python3 -c "
from pymongo import MongoClient
from datetime import datetime, timezone, timedelta
import os; from dotenv import load_dotenv; load_dotenv()
db = MongoClient(os.environ['MONGO_URL'])[os.environ['DB_NAME']]
db.users.insert_one({'user_id':'test_admin','email':'test.admin@example.com','name':'Test Admin','picture':'','is_admin':True,'created_at':datetime.now(timezone.utc).isoformat()})
db.user_sessions.insert_one({'user_id':'test_admin','session_token':'test_admin_token','expires_at':(datetime.now(timezone.utc)+timedelta(days=7)).isoformat(),'created_at':datetime.now(timezone.utc).isoformat()})
print('ok')
"
```

## Backend API tests (bearer)
```bash
BASE=https://shahjahanpur-news-1.preview.emergentagent.com
curl -s $BASE/api/auth/me -H "Authorization: Bearer test_admin_token"
curl -s -X POST $BASE/api/news -H "Authorization: Bearer test_admin_token" -H "Content-Type: application/json" -d '{"title":"t","category":"शाहजहांपुर"}'
curl -s -X POST $BASE/api/gallery -H "Authorization: Bearer test_admin_token" -H "Content-Type: application/json" -d '{"src":"https://x/a.jpg","caption":"c"}'
curl -s $BASE/api/videos      # public YouTube feed
curl -s $BASE/api/gallery     # public
curl -s $BASE/api/news        # public
```

## Browser test (cookie)
```python
await page.context.add_cookies([{
  "name":"session_token","value":"test_admin_token",
  "domain":"shahjahanpur-news-1.preview.emergentagent.com","path":"/",
  "httpOnly":True,"secure":True,"sameSite":"None"
}])
await page.goto("https://shahjahanpur-news-1.preview.emergentagent.com/admin")
# Should show Admin Panel (tabs: फोटो गैलरी / खबरें), not the login screen.
```

## CLEANUP (CRITICAL — so the real owner becomes first admin)
```bash
cd /app/backend && python3 -c "
from pymongo import MongoClient; import os; from dotenv import load_dotenv; load_dotenv()
db=MongoClient(os.environ['MONGO_URL'])[os.environ['DB_NAME']]
db.users.delete_many({'email':'test.admin@example.com'})
db.user_sessions.delete_many({'session_token':'test_admin_token'})
db.news.delete_many({'title':'t'}); db.gallery.delete_many({'caption':'c'})
print('admins left:', db.users.count_documents({'is_admin':True}))
"
```
