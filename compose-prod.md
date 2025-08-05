Here’s a **complete step‑by‑step guide** to build and run **ERPNext + HRMS** as a single stable Docker image (`erpnext-hrms:v15.72.3`) using the official `apps.json` approach.

---

## **1. Prepare `apps.json`**

Create a file `apps.json` on your host machine:

```json
[
  {
    "url": "https://github.com/frappe/erpnext",
    "branch": "version-15"
  },
  {
    "url": "https://github.com/frappe/hrms",
    "branch": "version-15"
  }
]
```

> **Note:**
>
> * If you need `payments`, add it as well (ERPNext depends on it).
> * If HRMS needs ERPNext, always list ERPNext first.

---

## **2. Generate Base64 String**

Encode `apps.json` into a Base64 string so Docker can consume it:

```bash
export APPS_JSON_BASE64=$(base64 -w 0 apps.json)
```

Verify it:

```bash
echo -n $APPS_JSON_BASE64 | base64 -d | jq
```

You should see the same JSON output.

---

## **3. Build Your Custom Image**

Faster, uses prebuilt base layers:

```bash
docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=erpnext-hrms:v15.72.3 \
  --file=images/layered/Containerfile .
```

## **4. Verify the Image**

Check that the image exists:

```bash
docker images | grep erpnext-hrms
```

Optional: run a shell inside the image to confirm apps:

```bash
docker run --rm -it erpnext-hrms:v15.72.3 bash
ls apps/
```

You should see `frappe`, `erpnext`, and `hrms`.

---

## **5. Update `docker-compose.yml`**

Replace **all service images** with your custom image:

```yaml
  backend:
    image: erpnext-hrms:v15.72.3
    ...

  frontend:
    image: erpnext-hrms:v15.72.3
    ...

  scheduler:
    image: erpnext-hrms:v15.72.3
    ...

  websocket:
    image: erpnext-hrms:v15.72.3
    ...

  queue-long:
    image: erpnext-hrms:v15.72.3
    ...

  queue-short:
    image: erpnext-hrms:v15.72.3
    ...
```

> You **don’t need `bench get-app`** or `create-site` with HRMS anymore.
> Instead, your `create-site` service can directly install apps:

```yaml
command: >
  wait-for-it -t 120 db:3306;
  wait-for-it -t 120 redis-cache:6379;
  wait-for-it -t 120 redis-queue:6379;
  bench new-site --mariadb-user-host-login-scope='%' \
    --admin-password=admin \
    --db-root-username=root \
    --db-root-password=admin \
    --install-app erpnext \
    --install-app hrms \
    --set-default frontend;
```

---

## **6. Start the Stack**

Run your updated stack:

```bash
docker compose up -d
```

Check logs:

```bash
docker compose logs -f backend
```

---