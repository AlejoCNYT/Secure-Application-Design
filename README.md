# README — Amazon Linux 2023 (AL2023) on EC2 via AWS CLI **+ Apache & Spring TLS**  
**International / Bilingual (EN/ES) — Console‑only workflow — CloudShell friendly**  
Author: You ✨ • Last updated: 2025‑10‑11 (America/Bogotá)

---

## 0) Purpose / Propósito

**EN.** This guide shows how to: (1) launch Amazon Linux 2023 on EC2 **entirely from CLI**, (2) configure **two servers** (Apache + Spring Boot) with **TLS on both**, (3) add **login with hashed passwords**, and (4) meet the **course rubric** with clear deliverables, screenshots, and GitHub documentation.

**ES.** Esta guía enseña cómo: (1) iniciar Amazon Linux 2023 en EC2 **solo por CLI**, (2) configurar **dos servidores** (Apache + Spring Boot) con **TLS en ambos**, (3) añadir **login con contraseñas hasheadas**, y (4) cumplir la **rúbrica del curso** con entregables, capturas y documentación en GitHub.

> **Architecture (high‑level) / Arquitectura (alto nivel)**  
```
 Client (HTML+JS)  <--HTTPS-->  Apache (EC2-A)  <--HTTPS-->  Spring Boot API (EC2-B)
             downloads files                 reverse proxy OR direct client → API over TLS
             (static site/app)               (ambos con certificados Let's Encrypt)
```

<img width="936" height="223" alt="imagen" src="https://github.com/user-attachments/assets/28d6b7f3-e966-47ab-bd9a-d02641042c4c" />

---

## 1) What we did / Lo que hicimos

- **EN.**
  - Created an **SSH key pair** and a **Security Group** in the **default VPC**.
  - Opened ports **22/80/443** as required (SSH from your IP; HTTP/HTTPS from everywhere for the demo).
  - Queried **SSM Parameter Store** for the **latest AL2023 AMI** (x86_64 or aarch64).
  - Launched EC2 instances from CLI and **captured the Instance IDs** robustly.
  - Ensured a **public subnet** has `MapPublicIpOnLaunch=true` to get a public IP.
  - Retrieved **Public IP/DNS**; connected via **SSH**.
  - Optional: installed **LAMP** (Apache/PHP/MariaDB).
  - Implemented **TLS**: Apache (EC2‑A) and Spring Boot (EC2‑B) with **Let's Encrypt**.
  - Implemented **login** with **BCrypt password hashing** in Spring Security.
  - Produced **cleanup** steps and **GitHub deliverables**.

- **ES.**
  - Creamos **par de claves SSH** y un **Security Group** en la **VPC por defecto**.
  - Abrimos puertos **22/80/443** según sea necesario (SSH desde tu IP; HTTP/HTTPS público para demo).
  - Consultamos **SSM Parameter Store** para la **última AMI de AL2023** (x86_64 o aarch64).
  - Lanzamos instancias EC2 por CLI y **capturamos los IDs** de forma robusta.
  - Aseguramos que la **subred pública** tenga `MapPublicIpOnLaunch=true` para obtener IP pública.
  - Obtenemos **IP/DNS públicos**; conexión por **SSH**.
  - Opcional: instalación **LAMP** (Apache/PHP/MariaDB).
  - **TLS** en Apache (EC2‑A) y Spring Boot (EC2‑B) con **Let's Encrypt**.
  - **Login** con **hash BCrypt** en Spring Security.
  - Pasos de **limpieza** y **entregables** para GitHub.

<img width="1877" height="675" alt="imagen" src="https://github.com/user-attachments/assets/57c1b210-7e0f-48bc-87bb-24ce46d82cf6" />

---

## 2) Prerequisites / Prerrequisitos

- AWS CLI v2 configured (`aws configure`) with EC2, SSM and VPC permissions.  
- Recommended: **AWS CloudShell** (already authenticated, with `jq`).  
- Disable the pager to avoid `less` screens:

```bash
export AWS_PAGER=""
```


<img width="1866" height="724" alt="imagen" src="https://github.com/user-attachments/assets/043478ea-e06b-4eb4-9ed2-f81a79025265" />

---

## 3) Set variables / Definir variables

```bash
# Region & project
export REGION="us-east-1"
export PROJECT="al2023-lab"

# Keys & security group
export KEY_NAME="${PROJECT}-key"
export SG_NAME="${PROJECT}-sg"

# Instance type & AMI (x86_64). For arm64/Graviton2, see below.
export INSTANCE_TYPE="t3.micro"                     # x86_64 (free-tier)
export SSM_PARAM="al2023-ami-kernel-default-x86_64" # latest x86_64 AL2023

# Arm64/Graviton2+ (optional)
# export INSTANCE_TYPE="t4g.micro"
# export SSM_PARAM="al2023-ami-kernel-default-arm64"
```


<img width="1866" height="724" alt="imagen" src="https://github.com/user-attachments/assets/043478ea-e06b-4eb4-9ed2-f81a79025265" />

---

## 4) Create SSH key / Crear clave SSH

**Bash (CloudShell/Linux/macOS):**
```bash
mkdir -p ~/.ssh
aws ec2 create-key-pair \
  --region "$REGION" \
  --key-name "$KEY_NAME" \
  --query 'KeyMaterial' --output text > ~/.ssh/${KEY_NAME}.pem
chmod 400 ~/.ssh/${KEY_NAME}.pem
ls -l ~/.ssh/${KEY_NAME}.pem
```

**Windows PowerShell (fuera de CloudShell):**
```powershell
$Region="us-east-1"; $Project="al2023-lab"; $KeyName="$Project-key"
New-Item -ItemType Directory -Force -Path "$HOME\.ssh" | Out-Null
$PemPath="$HOME\.ssh\$KeyName.pem"
aws ec2 create-key-pair --region $Region --key-name $KeyName `
  --query 'KeyMaterial' --output text | Out-File -FilePath $PemPath -Encoding ascii
icacls $PemPath /inheritance:r
icacls $PemPath /grant:r "$($env:USERNAME):(R)"
```

<img width="1859" height="691" alt="imagen" src="https://github.com/user-attachments/assets/5c3c5790-e721-4a14-a34b-9ce5e7f37aa2" />

---

## 5) Security Group in default VPC / SG en VPC por defecto

```bash
# Default VPC
VPC_ID=$(aws ec2 describe-vpcs --region "$REGION" \
  --filters Name=isDefault,Values=true \
  --query 'Vpcs[0].VpcId' --output text)

# Create SG
SG_ID=$(aws ec2 create-security-group --region "$REGION" \
  --group-name "$SG_NAME" \
  --description "SG for $PROJECT (22,80,443)" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' --output text)
echo "SG_ID=$SG_ID"

# Inbound rules
MY_IP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress --region "$REGION" \
  --group-id "$SG_ID" \
  --ip-permissions "IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges=[{CidrIp=${MY_IP}/32,Description='SSH from my IP'}]"

aws ec2 authorize-security-group-ingress --region "$REGION" \
  --group-id "$SG_ID" --protocol tcp --port 80  --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --region "$REGION" \
  --group-id "$SG_ID" --protocol tcp --port 443 --cidr 0.0.0.0/0
```

<img width="1871" height="463" alt="imagen" src="https://github.com/user-attachments/assets/35812d3f-bdbb-47d5-b508-307240682b88" />

---

## 6) Ensure public subnet / Asegurar subred pública

```bash
# Pick a public subnet (MapPublicIpOnLaunch=true)
read SUBNET_ID AZ <<<$(aws ec2 describe-subnets --region "$REGION" \
  --filters Name=vpc-id,Values="$VPC_ID" \
  --query 'Subnets[?MapPublicIpOnLaunch==`true`][0].{Id:SubnetId,Az:AvailabilityZone}' \
  --output text)

# If none found, enable public IP assignment on the first subnet
if [ -z "$SUBNET_ID" ] || [ "$SUBNET_ID" = "None" ]; then
  SUBNET_ID=$(aws ec2 describe-subnets --region "$REGION" \
    --filters Name=vpc-id,Values="$VPC_ID" \
    --query 'Subnets[0].SubnetId' --output text)
  aws ec2 modify-subnet-attribute --region "$REGION" \
    --subnet-id "$SUBNET_ID" --map-public-ip-on-launch
fi

echo "SUBNET_ID=$SUBNET_ID"
```

<img width="1000" height="299" alt="imagen" src="https://github.com/user-attachments/assets/8e3a8c5e-c5cc-4d85-88ad-a181c419f82a" />

---

## 7) Latest AL2023 AMI via SSM / Última AMI AL2023 vía SSM

```bash
AMI_ID=$(aws ssm get-parameters --region "$REGION" \
  --names "/aws/service/ami-amazon-linux-latest/${SSM_PARAM}" \
  --query 'Parameters[0].Value' --output text)
echo "AMI_ID=$AMI_ID"
```

<img width="779" height="360" alt="imagen" src="https://github.com/user-attachments/assets/0dc66d8c-47a4-407f-a0e9-1bf64455b517" />

---

## 8) Launch EC2 (robust) / Lanzar EC2 (robusto)

```bash
INSTANCE_ID=$(aws ec2 run-instances --region "$REGION" \
  --image-id "$AMI_ID" --count 1 --instance-type "$INSTANCE_TYPE" \
  --key-name "$KEY_NAME" --security-group-ids "$SG_ID" \
  --subnet-id "$SUBNET_ID" \
  --query 'Instances[0].InstanceId' --output text)

echo "INSTANCE_ID=$INSTANCE_ID"

# Waiters
aws ec2 wait instance-running   --region "$REGION" --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-status-ok --region "$REGION" --instance-ids "$INSTANCE_ID"

# Public info
PUB_IP=$(aws ec2 describe-instances --region "$REGION" \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
PUB_DNS=$(aws ec2 describe-instances --region "$REGION" \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicDnsName' --output text)
echo "PUB_IP=$PUB_IP  PUB_DNS=$PUB_DNS"
```

> **Tip.** If you ever see `Invalid id: "None"`, it means your `INSTANCE_ID` variable was empty; ensure the `run-instances` line is executed and its output captured before the waiters.

---

## 9) Connect & optional LAMP / Conectar e instalar LAMP (opcional)

```bash
ssh -i ~/.ssh/${KEY_NAME}.pem ec2-user@${PUB_DNS}
sudo dnf -y upgrade
sudo dnf -y install httpd php php-fpm php-mysqli php-json mariadb105-server
sudo systemctl enable --now httpd php-fpm
sudo systemctl enable --now mariadb
# Test PHP
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/phpinfo.php
```

**Secure MariaDB / Asegurar MariaDB**
```bash
sudo mysql_secure_installation
sudo mysql -u root -p -e "SELECT VERSION();"
```

<img width="1318" height="1022" alt="imagen" src="https://github.com/user-attachments/assets/4523e77c-d39c-4fc5-94cf-08ef3b4c6216" />

---

## 10) Apache TLS (EC2‑A) with Let’s Encrypt

> Replace `www.example.com` with your domain pointing to EC2‑A.

```bash
sudo dnf -y install certbot python3-certbot-apache || true

# If the apache plugin isn't available, you can use standalone and then copy:
# sudo systemctl stop httpd
# sudo certbot certonly --standalone -d www.example.com -m you@example.com --agree-tos --non-interactive
# sudo systemctl start httpd

# Preferred (Apache plugin):
sudo certbot --apache -d www.example.com -m you@example.com --agree-tos --redirect --non-interactive

# Verify renewal timer
systemctl list-timers | grep certbot || true
```

**Result:** Static client (HTML+JS) is now served via **HTTPS** from Apache.

---

## 11) Spring Boot TLS (EC2‑B) with Let’s Encrypt

> Use a second domain, e.g., `api.example.com`, pointing to EC2‑B.  
> If you run Nginx/Apache as a reverse proxy to Spring Boot, you can reuse standard `certbot --nginx/--apache`.  
> For **direct Spring TLS**, create a **PKCS#12** keystore from the LE certs and configure Spring.

```bash
# Obtain certificate (standalone)
sudo dnf -y install certbot
sudo systemctl stop firewalld 2>/dev/null || true
sudo certbot certonly --standalone -d api.example.com -m you@example.com --agree-tos --non-interactive

# Convert to PKCS#12 for Spring
DOMAIN=api.example.com
sudo openssl pkcs12 -export \
  -in  /etc/letsencrypt/live/$DOMAIN/fullchain.pem \
  -inkey /etc/letsencrypt/live/$DOMAIN/privkey.pem \
  -out /opt/spring/spring.p12 -name tomcat \
  -password pass:changeit
sudo chown ec2-user:ec2-user /opt/spring/spring.p12
```

**Minimal `application.properties`:**
```properties
server.port=8443
server.ssl.enabled=true
server.ssl.key-store=/opt/spring/spring.p12
server.ssl.key-store-type=PKCS12
server.ssl.key-store-password=changeit
```

**Systemd service (`/etc/systemd/system/spring.service`):**
```ini
[Unit]
Description=Spring Boot API
After=network.target

[Service]
User=ec2-user
Environment=SPRING_CONFIG_LOCATION=/opt/spring/application.properties
ExecStart=/usr/bin/java -jar /opt/spring/app.jar
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now spring
sudo systemctl status spring --no-pager | head -n 20
```

**Client → API over TLS:** call `https://api.example.com:8443/...` (or proxy to 443).

---

## 12) Login (BCrypt) / Inicio de sesión (BCrypt)

**Spring Security snippet (Kotlin/Java pseudo):**
```java
@Bean
PasswordEncoder passwordEncoder(){ return new BCryptPasswordEncoder(); }

// When creating users:
String hash = passwordEncoder().encode(plainPassword);
// store 'hash' (never the plain text)

// Auth compares using passwordEncoder.matches(raw, hash).
```

**Tip.** Prefer **argon2id** or **BCrypt** with a strong factor; never store plain passwords.

---

## 13) Cleanup / Limpieza

```bash
# Terminate instance(s)
aws ec2 terminate-instances --region "$REGION" --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-terminated --region "$REGION" --instance-ids "$INSTANCE_ID"

# Release Elastic IP if used
# aws ec2 release-address --region "$REGION" --allocation-id "$ALLOC_ID"

# Delete SG & KeyPair
aws ec2 delete-security-group --region "$REGION" --group-id "$SG_ID"
aws ec2 delete-key-pair        --region "$REGION" --key-name "$KEY_NAME"
rm -f ~/.ssh/${KEY_NAME}.pem
```

<img width="1288" height="483" alt="imagen" src="https://github.com/user-attachments/assets/77fe4e66-db8d-4b6b-a01f-edda0a5171a5" />

---

## 14) Rubric mapping / Mapeo a la rúbrica

### Class Work (50%)
- **Participation & Collaboration (20%)** — Use the **discussion prompts** below to drive design choices; contribute code via PRs.
- **Hands‑on Lab Performance (30%)**
  - Deploy **two EC2 instances** (Apache + Spring) using this README.
  - **Apache** serves the client; **Spring** exposes HTTPS REST.
  - **TLS** configured on **both**; sample **login** uses **hashed passwords**.
  - **Let’s Encrypt** certs generated/installed on both servers.
  - **All code + README + screenshots** pushed to GitHub.

### Homework (50%)
- **Application Architecture Design (25%)**
  - Include a **design doc** (one‑pager) in your repo explaining the topology, domains, ports, and trust boundaries.
- **Security Implementation (15%)**
  - Demo video hitting **HTTPS endpoints** end‑to‑end; show **BCrypt** hashes and denied access on wrong creds.
- **Final Deliverables (10%)**
  - **GitHub repo** with source code, this **README**, the **architecture overview**, and **screenshots**.
  - **Short video**: deployment + explanation of security features.

---

## 17) Troubleshooting / Solución de problemas

- `Invalid id: "None"` → variable vacía: asegúrate de capturar `INSTANCE_ID` del `run-instances`.  
- No public IP → valida `MapPublicIpOnLaunch=true` en la subred y que **no** uses una **interfaz solo privada**.  
- Certbot plugin missing → usa **standalone** o un **proxy**; luego convierte a **PKCS#12** para Spring.  
- 403/401 con Spring → revisa CORS, roles y el `PasswordEncoder` (usa `matches()` con el hash).

---

## License / Licencia
MIT (if you want). Replace as needed.



