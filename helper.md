Docker Purge all

echo "==> Stopping and removing existing containers..."
sudo docker compose --project-name erpnextlab -f ~/gitops/docker-compose.yml down --volumes --remove-orphans

sudo docker compose -f ~/frappe_docker/pwd.yml down --volumes --remove-orphans

echo "==> Pruning all unused Docker data (containers, images, networks, volumes)..."
sudo docker system prune -a --volumes -f



Frappe-docker Create Compose file from all overrides (no proxy, postgres)
 
sudo docker compose -f ~/frappe_docker/compose.yaml   -f ~/frappe_docker/overrides/compose.noproxy.yaml   -f ~/frappe_docker/overrides/compose.postgres.yaml   -f ~/frappe_docker/overrides/compose.redis.yaml   config > ~/gitops/docker-compose.yml

Create .env file

# Reference: https://github.com/frappe/frappe_docker/blob/main/docs/environment-variables.md
ERPNEXT_VERSION=v15.70.0 
DB_PASSWORD=admin
LETSENCRYPT_EMAIL=mail@example.com
SITES=`erpnextlab.itretina.com`

Option 1 Install erpnext native of the docker image
 
Frappe-docker create new site if not present already (must create when installing erpnext and frappe framework for the first time)
 
sudo docker compose --project-name erpnext -f ~/gitops/docker-compose.yml exec backend   bench new-site erpnextlab.itretina.com --db-type postgres --db-host db --db-port 5432 --db-root-username postgres --db-root-password admin --admin-password admin

Install erpnext (already present in the docker image no need to bench get-app)

sudo docker compose --project-name erpnext -f ~/gitops/docker-compose.yml exec backend   bench --site erpnext.itretina.com install-app erpnext

Option 2 Install erpnext latest with hrms

sudo docker compose -f pwd.yml exec backend bash
bench get-app --branch version-15 hrms https://github.com/frappe/hrms.git
bench --site erpnextlab.itretina.com install-app hrms


echo "==> Installing ERPNext + HRMS (version-15)..."
sudo docker compose --project-name $PROJECT_NAME -f $COMPOSE_FILE exec backend bash -c "
set -e
cd /home/frappe/frappe-bench

# Fetch compatible apps
bench get-app --branch version-15 erpnext https://github.com/frappe/erpnext.git
bench get-app --branch version-15 hrms https://github.com/frappe/hrms.git

# Create a new site
bench new-site erpnextlab.itretina.com --db-type postgres --db-host db --db-port 5432 --db-root-username postgres --db-root-password admin --admin-password 'admin'

# Install apps in correct order
bench --site $SITE_NAME install-app erpnext
bench --site $SITE_NAME install-app hrms

# Run final migration to ensure all patches apply
bench --site $SITE_NAME migrate
"


Start all

sudo docker compose --project-name erpnextlab -f ~/gitops/docker-compose.yml up -v
 
Stop all

sudo docker compose --project-name erpnextlab -f ~/gitops/docker-compose.yml down -v
 


--------------

Frappe-docker Create Compose file from all overrides (no proxy, mariadb)

sudo docker compose -f ~/frappe_docker/compose.yaml   -f ~/frappe_docker/overrides/compose.noproxy.yaml   -f ~/frappe_docker/overrides/compose.mariadb.yaml   -f ~/frappe_docker/overrides/compose.redis.yaml   config > ~/gitops/docker-compose.yml

Frappe-docker create new site if not present already (must create when installing erpnext and frappe framework for the first time)

sudo docker compose --project-name erpnextlab -f ~/gitops/docker-compose.yml exec backend   bench new-site erpnextlab.itretina.com --db-type mariadb --db-host db --db-port 3306 --db-root-username root --db-root-password admin --admin-password 'admin'

sudo docker compose --project-name erpnextlab -f ~/gitops/docker-compose.yml down -v

sudo docker compose --project-name erpnextlab -f ~/gitops/docker-compose.yml up -v



-----
Check logs

sudo docker compose --project-name erpnextlab -f ~/gitops/docker-compose.yml logs backend -f
sudo docker compose -f pwd.yml logs backend -f



-----
Working on our own compose file

sudo docker compose -f compose-prod.yaml down -v

When dockerfile changes
sudo docker compose -f compose-prod.yaml up -d --build


sudo docker compose -f compose-prod.yaml up -d --build --force-recreate


sudo docker compose -f compose-prod.yaml up -d --force-recreate
sudo docker compose -f compose-prod.yaml up -d
sudo docker compose -f compose-prod.yaml logs -f
sudo docker compose -f compose-prod.yaml logs -f configurator


Bench access
sudo docker compose -f compose-prod.yaml exec backend bash
