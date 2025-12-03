# SQL Server on Docker with Tailscale - Complete Setup Guide
Bu sənəd Ubuntu-da Docker konteynerində SQL Server quraşdırılması və Tailscale ilə hər yerdən təhlükəsiz qoşulma prosesinı əhatə edir.

![Ubuntu](https://img.shields.io/badge/Linux-Ubuntu-orange)
![Docker](https://img.shields.io/badge/Docker-Containerization-lightblue)
![SQLServer](https://img.shields.io/badge/SQLServer-blue)
![TailScale](https://img.shields.io/badge/TailScale-yellow)

## 📋 İçindəkilər:

1. **Tələblər**
2. **Docker Quraşdırılması**
3. **SQL Server Konteynerinin Yaradılması**
4. **Tailscale Quraşdırılması**
5. **SQL Server İstifadəçi İdarəetməsi**
6. **Qoşulma və Test**
7. **Faydalı Əmrlər**


## Tələblər

- **Ubuntu 20.04 və ya daha yeni versiya**
- **Minimum 2GB RAM**
- **10GB boş disk sahəsi**
- **İnternet bağlantısı**
- **sudo icazələri**


## Docker Quraşdırılması

1. Docker APT repository quraşdırın
```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

2. Docker paketlərini quraşdırın
```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

3. Docker-in işlədiyini yoxlayın
```bash
# Status yoxlayın
sudo systemctl status docker

# Lazım gələrsə işə salın
sudo systemctl start docker

# Test edin
sudo docker run hello-world
```

## SQL Server Konteynerinin Yaradılması

**1. Docker Compose faylı yaradın**

Layihə qovluğu yaradın:
```bash
mkdir -p ~/sql-server
cd ~/sql-server
```
docker-compose.yml faylı yaradın:

```bash
nano docker-compose.yml
```

Aşağıdakı məzmunu əlavə edin:
```bash
services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: sqlserver
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong@Password123
      - MSSQL_PID=Developer
    ports:
      - "1433:1433"
    volumes:
      - sqlserver_data:/var/opt/mssql
    restart: unless-stopped
    networks:
      - sql_network

volumes:
  sqlserver_data:
    driver: local

networks:
  sql_network:
    driver: bridge
```

⚠️ Vacib: SA_PASSWORD dəyərini güclü şifrə ilə dəyişdirin!

**2. Konteyneri işə salın**
```bash
sudo docker compose up -d
```

**3. SQL Server-in hazır olduğunu yoxlayın**
```bash
# Statusu yoxlayın
sudo docker compose ps

# Logları izləyin
sudo docker compose logs -f sqlserver
```

"SQL Server is now ready for client connections" mesajını gözləyin (10-20 saniyə).

## Tailscale Quraşdırılması

Tailscale fərqli şəbəkələrdən təhlükəsiz qoşulma üçün istifadə olunur.

**1. Ubuntu-da (SQL Server VM)**
```bash
#Tailscale quraşdırın
curl -fsSL https://tailscale.com/install.sh | sh

# İşə salın
sudo tailscale up

# IP ünvanını öyrənin
tailscale ip -4
IP ünvanını qeyd edin (məsələn: 100.64.x.x)
```

**2. Digər cihazlarda**
Windows / Mac / Linux:

https://tailscale.com/download saytından yükləyin
Quraşdırın və eyni hesabla giriş edin

Android / iOS:

App Store və ya Google Play-dən Tailscale yükləyin
Eyni hesabla giriş edin


SQL Server İstifadəçi İdarəetməsi
SQL Server Management Studio (SSMS) ilə

SSMS-də qoşulun:

Server: localhost (lokal) və ya Tailscale IP
Authentication: SQL Server Authentication
Username: sa
Password: docker-compose.yml-də təyin etdiyiniz şifrə


New Query açın və aşağıdakı SQL-i çalışdırın:

sql-- Yeni istifadəçi yaradın
CREATE LOGIN khayyam WITH PASSWORD = 'Khayyam@2025!Strong';
GO

-- Sysadmin icazəsi verin (tam icazə)
ALTER SERVER ROLE sysadmin ADD MEMBER khayyam;
GO

-- Master database
USE master;
GO
CREATE USER khayyam FOR LOGIN khayyam;
GO

-- AdventureWorks2022 (əgər varsa)
USE AdventureWorks2022;
GO
CREATE USER khayyam FOR LOGIN khayyam;
GO
ALTER ROLE db_owner ADD MEMBER khayyam;
GO

-- İcazələri yoxlayın
SELECT 
    p.name AS username,
    r.name AS role_name
FROM sys.server_role_members srm
JOIN sys.server_principals p ON srm.member_principal_id = p.principal_id
JOIN sys.server_principals r ON srm.role_principal_id = r.principal_id
WHERE p.name = 'khayyam';
GO
Terminaldan (sqlcmd ilə)
bash# sqlcmd quraşdırın
sudo apt install mssql-tools unixodbc-dev

# SQL Server-ə qoşulun
/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourStrong@Password123'

# Yuxarıdakı SQL əmrlərini çalışdırın

Qoşulma və Test
Eyni Şəbəkədən (Eyni Wi-Fi/LAN)
Qoşulma məlumatları:

Host: Ubuntu VM-in lokal IP-si (məsələn: 192.168.1.100)
Port: 1433
Username: khayyam və ya sa
Password: təyin etdiyiniz şifrə

Fərqli Şəbəkədən (Tailscale ilə)
Qoşulma məlumatları:

Host: Tailscale IP (məsələn: 100.64.x.x)
Port: 1433
Username: khayyam və ya sa
Password: təyin etdiyiniz şifrə

Test sorğuları
sql-- Database-ləri siyahıya alın
SELECT name FROM sys.databases;
GO

-- Yeni database yaradın
CREATE DATABASE TestDB;
GO

-- Database-ə keçin
USE TestDB;
GO

-- Test table yaradın
CREATE TABLE Users (
    ID INT PRIMARY KEY IDENTITY(1,1),
    Name NVARCHAR(100),
    Email NVARCHAR(100)
);
GO

-- Data əlavə edin
INSERT INTO Users (Name, Email) VALUES ('Test User', 'test@example.com');
GO

-- Data oxuyun
SELECT * FROM Users;
GO

Faydalı Əmrlər
Docker Compose əmrləri
bash# Konteyneri işə salın
sudo docker compose up -d

# Dayanın
sudo docker compose down

# Yenidən başladın
sudo docker compose restart

# Statusu yoxlayın
sudo docker compose ps

# Logları görün
sudo docker compose logs -f sqlserver

# Container-ə daxil olun
sudo docker exec -it sqlserver /bin/bash
SQL Server əmrləri (Container içində)
bash# sqlcmd ilə qoşulun
sudo docker exec -it sqlserver /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourStrong@Password123'

# Database-ləri görün (birbaşa)
sudo docker exec -it sqlserver /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourStrong@Password123' -Q "SELECT name FROM sys.databases;"
Tailscale əmrləri
bash# Status yoxlayın
tailscale status

# IP ünvanını görün
tailscale ip -4

# Yenidən qoşulun
sudo tailscale up

# Çıxış edin
sudo tailscale down
Backup və Restore
Backup
sql-- Database backup
BACKUP DATABASE TestDB 
TO DISK = '/var/opt/mssql/data/TestDB.bak'
WITH FORMAT, COMPRESSION;
GO
bash# Backup faylını çıxarın
sudo docker cp sqlserver:/var/opt/mssql/data/TestDB.bak ~/TestDB.bak
Restore
bash# Backup faylını konteynerə köçürün
sudo docker cp ~/TestDB.bak sqlserver:/var/opt/mssql/data/
sql-- Database restore
RESTORE DATABASE TestDB
FROM DISK = '/var/opt/mssql/data/TestDB.bak'
WITH REPLACE;
GO

Firewall Konfiqurasiyası (Lazım olsa)
bash# UFW ilə port 1433-ü açın
sudo ufw allow 1433/tcp
sudo ufw reload

# UFW statusunu yoxlayın
sudo ufw status

Problemlərin Həlli
Container işləmir
bash# Logları yoxlayın
sudo docker compose logs sqlserver

# Yenidən başladın
sudo docker compose restart
Qoşulma problemi

Firewall yoxlayın:

bash   sudo ufw status

SQL Server hazırdırmı:

bash   sudo docker compose logs -f sqlserver | grep "ready"

Port açıqdırmı:

bash   sudo netstat -tulpn | grep 1433
Şifrə dəyişmək
sqlALTER LOGIN sa WITH PASSWORD = 'NewStrong@Password456';
GO

Təhlükəsizlik Tövsiyələri

✅ Güclü şifrələr istifadə edin (minimum 8 simvol, böyük/kiçik hərf, rəqəm, simvol)
✅ sa istifadəçisi əvəzinə fərqli istifadəçilər yaradın
✅ Firewall konfiqurasiyasını düzgün təyin edin
✅ Mütəmadi backup alın
✅ SQL Server-i güncel saxlayın
✅ Tailscale istifadə edərək internetdən birbaşa port açmayın


Əlavə Resurslar

Docker Documentation
SQL Server on Linux
Tailscale Documentation
SQL Server Best Practices


Lisenziya
Bu sənəd MIT lisenziyası altında paylaşılır.

Müəllif: Khayyam
Tarix: Dekabr 2024
Versiya: 1.0