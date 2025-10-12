# README — Amazon Linux 2023 (AL2023) on EC2 via AWS CLI
**International / Bilingual (EN/ES)** — **Console-only workflow** — **CloudShell friendly**  
Author: You ✨ • Last updated: 2025-10-02 (America/Bogotá)

---

## 1) What we did / Lo que hicimos
<img width="936" height="223" alt="imagen" src="https://github.com/user-attachments/assets/28d6b7f3-e966-47ab-bd9a-d02641042c4c" />

- **Created an SSH key pair** to access the instance.
- **Created a Security Group** (SG) in the **default VPC**.
- Opened **SSH (22)** only from your **public IP**, and **HTTP (80)** and **HTTPS (443)** from everywhere.
- **Fetched the latest AL2023 AMI** dynamically using **SSM Parameter Store**.
- **Launched an EC2 instance** via CLI. We learned to **capture the Instance ID** correctly and to **force a public IP** via network interface.
- Handled issues: `Invalid id: "None"` (empty variable), `wait instance-running ... shutting-down` (instance terminated).
- Provided steps to **wait for health**, **retrieve Public IP/DNS**, **SSH**, **install LAMP** (optional), and **cleanup**.
  
> **Tip:** This manual uses **`$REGION=us-east-1`**, project name `al2023-lab`, and **x86_64** (`t3.micro`). For Graviton (**arm64**), switch to `t4g.micro` and the corresponding SSM AMI parameter.

---

## 2) Prerequisites / Prerrequisitos

- **AWS CLI v2** configured (`aws configure`) with permissions for **EC2**, **SSM** y **VPC**.  
- Recomendado: **AWS CloudShell** (ya autenticado y con `jq`).  
- Desactivar el pager del CLI (para evitar pantallas de `less`):

```bash
export AWS_PAGER=""
```

---

## 3) Set variables / Definir variables
<img width="1877" height="675" alt="imagen" src="https://github.com/user-attachments/assets/57c1b210-7e0f-48bc-87bb-24ce46d82cf6" />
<img width="1866" height="724" alt="imagen" src="https://github.com/user-attachments/assets/043478ea-e06b-4eb4-9ed2-f81a79025265" />

```bash
# Region & project
export REGION="us-east-1"
export PROJECT="al2023-lab"

# Keys & security group
export KEY_NAME="${PROJECT}-key"
export SG_NAME="${PROJECT}-sg"

# Instance type & AMI (x86_64). For arm64, see below.
export INSTANCE_TYPE="t3.micro"                          # x86_64 (free-tier)
export SSM_PARAM="al2023-ami-kernel-default-x86_64"      # latest x86_64 AL2023

# Arm64/Graviton2+ (optional):
# export INSTANCE_TYPE="t4g.micro"
# export SSM_PARAM="al2023-ami-kernel-default-arm64"
```

---

## 4) Create SSH key / Crear clave SSH

```bash
mkdir -p ~/.ssh
aws ec2 create-key-pair \
  --region "$REGION" \
  --key-name "$KEY_NAME" \
  --query 'KeyMaterial' --output text > ~/.ssh/${KEY_NAME}.pem

chmod 400 ~/.ssh/${KEY_NAME}.pem
ls -l ~/.ssh/${KEY_NAME}.pem
```

> **Windows PowerShell (si usas Windows fuera de CloudShell):**
```powershell
$Region="us-east-1"; $Project="al2023-lab"; $KeyName="$Project-key"
New-Item -ItemType Directory -Force -Path "$HOME\.ssh" | Out-Null
$PemPath="$HOME\.ssh\$KeyName.pem"
aws ec2 create-key-pair --region $Region --key-name $KeyName `
  --query 'KeyMaterial' --output text | Out-File -FilePath $PemPath -Encoding ascii
icacls $PemPath /inheritance:r
icacls $PemPath /grant:r "$($env:USERNAME):(R)"
```

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

---

## 6) Ensure public subnet / Asegurar subred pública
<img width="1859" height="691" alt="imagen" src="https://github.com/user-attachments/assets/5c3c5790-e721-4a14-a34b-9ce5e7f37aa2" />
<img width="1871" height="463" alt="imagen" src="https://github.com/user-attachments/assets/35812d3f-bdbb-47d5-b508-307240682b88" />
<img width="1000" height="299" alt="imagen" src="https://github.com/user-attachments/assets/8e3a8c5e-c5cc-4d85-88ad-a181c419f82a" />

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

<img width="779" height="360" alt="imagen" src="https://github.com/user-attachments/assets/0dc66d8c-47a4-407f-a0e9-1bf64455b517" />
<img width="1318" height="1022" alt="imagen" src="https://github.com/user-attachments/assets/4523e77c-d39c-4fc5-94cf-08ef3b4c6216" />

```

---
<img width="1288" height="483" alt="imagen" src="https://github.com/user-attachments/assets/77fe4e66-db8d-4b6b-a01f-edda0a5171a5" />

## 7) Latest AL2023 AMI via SSM / Última AMI AL2023 vía SSM

```bash
AMI_ID=$(aws ssm get-parameters --region "$REGION" \
  --names "/aws/service/ami-amazon-linux-latest/${SSM_PARAM}" \
  --query 'Parameters[0].Value' --output text)
echo "AMI_ID=$AMI_ID"
```

---

## 8) Run instance (force Public IP) / Lanzar instancia (forzar IP pública)

```bash
INSTANCE_ID=$(aws ec2 run-instances --region "$REGION" \
  --image-id "$AMI_ID" \
  --instance-type "$INSTANCE_TYPE" \
  --key-name "$KEY_NAME" \
  --network-interfaces "DeviceIndex=0,AssociatePublicIpAddress=true,SubnetId=${SUBNET_ID},Groups=${SG_ID}" \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${PROJECT}}]" \
  --query 'Instances[0].InstanceId' --output text)

echo "INSTANCE_ID=$INSTANCE_ID"
```

> **Why this matters / Por qué importa:** If you saw `Invalid id: "None"`, it means `$INSTANCE_ID` was empty. Always **capture** it directly from `run-instances` as above.

(Optional) Protect from accidental termination:
```bash
aws ec2 modify-instance-attribute --region "$REGION" \
  --instance-id "$INSTANCE_ID" --disable-api-termination
```

---

## 9) Wait and retrieve IP/DNS / Esperar y obtener IP/DNS

```bash
aws ec2 wait instance-running   --region "$REGION" --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-status-ok --region "$REGION" --instance-ids "$INSTANCE_ID"

PUBLIC_IP=$(aws ec2 describe-instances --region "$REGION" --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
PUBLIC_DNS=$(aws ec2 describe-instances --region "$REGION" --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicDnsName' --output text)
echo "PUBLIC_IP=$PUBLIC_IP"
echo "PUBLIC_DNS=$PUBLIC_DNS"
```

**Fallback (Elastic IP):**
```bash
if [ -z "$PUBLIC_IP" ] || [ "$PUBLIC_IP" = "None" ]; then
  ALLOC_ID=$(aws ec2 allocate-address --region "$REGION" --domain vpc \
    --query 'AllocationId' --output text)
  aws ec2 associate-address --region "$REGION" \
    --instance-id "$INSTANCE_ID" --allocation-id "$ALLOC_ID" \
    --query 'AssociationId' --output text
  PUBLIC_IP=$(aws ec2 describe-instances --region "$REGION" --instance-ids "$INSTANCE_ID" \
    --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
  echo "Elastic IP assigned: $PUBLIC_IP"
fi
```

---

## 10) SSH login / Conexión SSH

```bash
ssh -o StrictHostKeyChecking=no -i ~/.ssh/${KEY_NAME}.pem ec2-user@$PUBLIC_IP
# or / o
# ssh -o StrictHostKeyChecking=no -i ~/.ssh/${KEY_NAME}.pem ec2-user@$PUBLIC_DNS
```

> **Default user:** `ec2-user` (Amazon Linux).  
> If you get “Permission denied”, verify `chmod 400` on the key file and SG rule for port 22 to **your public IP**.

---

## 11) (Optional) LAMP quick setup inside the instance
**EN:** Run these after SSH. **ES:** Ejecuta estos pasos después de entrar por SSH.

```bash
# Update base
sudo dnf -y update

# Install Apache + PHP + MariaDB
sudo dnf -y install httpd php php-cli php-fpm php-mysqlnd php-opcache php-gd php-xml php-mbstring mariadb105-server

# Enable & start
sudo systemctl enable --now httpd
sudo systemctl enable --now mariadb

# Test PHP
echo '<?php phpinfo(); ?>' | sudo tee /var/www/html/info.php > /dev/null
sudo systemctl restart httpd

# Access:
#   http://<PUBLIC_IP>/
#   http://<PUBLIC_IP>/info.php   (remove after testing)
# Hardening (interactive):
#   sudo mysql_secure_installation
```

---

## 12) Troubleshooting / Solución de problemas

### A) `InvalidInstanceID.Malformed: Invalid id: "None"`
- **Cause:** `$INSTANCE_ID` was empty (often due to filtering by tag too early, or not capturing output).  
- **Fix:** Always capture from `run-instances`:
  ```bash
  INSTANCE_ID=$(aws ec2 run-instances ... --query 'Instances[0].InstanceId' --output text)
  ```
  Disable pager:
  ```bash
  export AWS_PAGER=""
  ```

### B) Waiter failed with `"shutting-down"`
- **Meaning:** The instance terminated before reaching `running` (user-initiated, capacity or launch issue).
- **Diagnose:**
  ```bash
  aws ec2 describe-instances --region "$REGION" --instance-ids "$INSTANCE_ID" \
    --query 'Reservations[0].Instances[0].{State:State.Name,Reason:StateTransitionReason,Msg:StateReason.Message,Az:Placement.AvailabilityZone}'

  aws ec2 describe-instance-status --region "$REGION" --instance-ids "$INSTANCE_ID" \
    --include-all-instances \
    --query 'InstanceStatuses[0].{Sys:InstanceStatus.Status,Inst:InstanceState.Name,Events:Events}'
  ```
- **Retry:** Launch again in a **public subnet** with `AssociatePublicIpAddress=true`. If capacity, choose a **different AZ** (another subnet).
<img width="1099" height="492" alt="imagen" src="https://github.com/user-attachments/assets/93132393-ff39-421a-8c02-23351f302737" />

### C) No public IP assigned
- Use a **public subnet** (MapPublicIpOnLaunch=true) or the **network-interfaces** block with `AssociatePublicIpAddress=true`.  
- As fallback, **allocate + associate** an **Elastic IP** (see step 9).

### D) SSH “Permission denied”
- Ensure correct permissions on key: `chmod 400 ~/.ssh/${KEY_NAME}.pem`.  
- Ensure SG rule for port 22 includes **your** public IP.  
- User is **ec2-user** for AL2023.

### E) Pager showed “LESS help”
- Set: `export AWS_PAGER=""`.

---

## 13) Cleanup / Limpieza
<img width="830" height="732" alt="imagen" src="https://github.com/user-attachments/assets/5e3d3180-27e9-4b73-a16e-a3f455f4b3cd" />
<img width="650" height="439" alt="imagen" src="https://github.com/user-attachments/assets/1652ed3d-2898-4f9c-a05b-09b376203f92" />

```bash
# Terminate instance
aws ec2 terminate-instances --region "$REGION" --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-terminated --region "$REGION" --instance-ids "$INSTANCE_ID"

# Release Elastic IP if used
# aws ec2 release-address --region "$REGION" --allocation-id "$ALLOC_ID"

# Delete SG & KeyPair
aws ec2 delete-security-group --region "$REGION" --group-id "$SG_ID"
aws ec2 delete-key-pair --region "$REGION" --key-name "$KEY_NAME"
rm -f ~/.ssh/${KEY_NAME}.pem
```

---

## 14) Notes / Notas

- **Architectures:** x86_64 → `t3.* / m5.*` and `al2023-ami-*-x86_64`.  
  Graviton (arm64) → `t4g.* / m7g.*` and `al2023-ami-*-arm64`. **AL2023 no soporta A1**.
- **Security best practice:** Lock SSH to your IP. Consider **AWS Systems Manager Session Manager** (no port 22) by attaching an IAM role with `AmazonSSMManagedInstanceCore` to the instance.
- **Remove test file:** `sudo rm -f /var/www/html/info.php` after testing.
- **Tagging:** Consistent tags (e.g., `Name=${PROJECT}`) help you filter instances later safely.

---

## 15) Optional: CloudFormation (short)

**Template (`al2023.yml`):**
```yaml
Parameters:
  LatestAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64'
Resources:
  Instance:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: 't3.micro'
      ImageId: !Ref LatestAmiId
      Tags:
        - Key: Name
          Value: al2023-lab
```

**Deploy:**
```bash
aws cloudformation deploy --region "$REGION" --template-file al2023.yml \
  --stack-name al2023-stack --capabilities CAPABILITY_NAMED_IAM
```
<img width="1269" height="690" alt="imagen" src="https://github.com/user-attachments/assets/2a06d82e-ccd3-4368-8c6f-6aff0ce43a5e" />
<img width="1292" height="663" alt="imagen" src="https://github.com/user-attachments/assets/3453dafa-7f75-4570-bf88-b02b4cedcbb1" />
<img width="945" height="248" alt="imagen" src="https://github.com/user-attachments/assets/ec218b4f-f098-4740-8ddd-d9451870166e" />

---

### End / Fin ✅
This manual reflects exactly what we set up and the issues we addressed (ID `None`, `shutting-down`, public IP assignment) so you can reproduce or adapt it quickly in future labs.
