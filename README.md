# SQL Server on Docker with Tailscale
![Ubuntu](https://img.shields.io/badge/Linux-Ubuntu-orange)
![Docker](https://img.shields.io/badge/Docker-Containerization-lightblue)
![SQLServer](https://img.shields.io/badge/SQLServer-blue)
![TailScale](https://img.shields.io/badge/TailScale-yellow)

<img width="2816" height="1504" alt="Gemini_Generated_Image_d4munjd4munjd4mu" src="https://github.com/user-attachments/assets/8f53cac9-e0c2-4ebb-8628-d24ce1366862" />


Ubuntu üzərində Docker konteynerində SQL Server-in quraşdırılması və Tailscale vasitəsilə müxtəlif şəbəkələrdən təhlükəsiz şəkildə qoşulma prosesini izah edir. Mobil cihazlardan qoşulmaq da mümkündür.

## 📋 İçindəkilər:

1. **[Tələblər](#tələblər)**
2. **[Docker Quraşdırılması](#docker-quraşdırılması)**
3. **[SQL Server Konteynerinin Yaradılması](#sql-server-konteynerinin-yaradılması)**
4. **[Tailscale Quraşdırılması](#tailscale-quraşdırılması)**
5. **[SQL Server İstifadəçi İdarəetməsi](#sql-server-i̇stifadəçi-i̇darəetməsi)**
6. **[Qoşulma və Test](#qoşulma-və-test)**
7. **[Faydalı Əmrlər](#faydalı-əmrlər)**
8. **[Təhlükəsizlik Tövsiyələri](#təhlükəsizlik-tövsiyələri)**
9. **[Əlavə Resurslar](#əlavə-resurslar)**

## Tələblər

- **Ubuntu 20.04 və ya daha yeni versiya**
- **Minimum 2GB RAM**
- **10GB boş disk sahəsi**
- **İnternet bağlantısı**
- **sudo icazələri**

---

## Docker Quraşdırılması

**1. Docker APT repository quraşdırın**
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

**2. Docker paketlərini quraşdırın**
```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**3. Docker-in işlədiyini yoxlayın**
- Status yoxlayın
```bash
sudo systemctl status docker
```

- İşə salın
```bash
sudo systemctl start docker
```

- Test edin
```bash
sudo docker run hello-world
```

---

## SQL Server Konteynerinin Yaradılması

**1. Docker Compose faylı yaradın**

- Layihə qovluğu yaradın:
```bash
mkdir -p ~/sql-server
cd ~/sql-server
```
- docker-compose.yml faylı yaradın:

```bash
nano docker-compose.yml
```

- Aşağıdakı məzmunu əlavə edin:
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

- "SQL Server is now ready for client connections" mesajını gözləyin (10-20 saniyə).

---

## Tailscale Quraşdırılması

- Tailscale fərqli şəbəkələrdən təhlükəsiz qoşulma üçün istifadə olunur.

**1. Ubuntu-da (SQL Server VM)**

- Tailscale quraşdırın
```bash
curl -fsSL https://tailscale.com/install.sh | sh
```
- İşə salın
```bash
sudo tailscale up
```
- IP ünvanını öyrənin
```bash
tailscale ip -4
```
- IP ünvanını qeyd edin (məsələn: 100.64.x.x)

**2. Digər cihazlarda**
**Windows / Mac / Linux:**

1. [Tailscale](https://tailscale.com/download) saytından yükləyin
2. Quraşdırın və eyni hesabla giriş edin

**Android / iOS:**

1. App Store və ya Google Play-dən Tailscale yükləyin
2. Eyni hesabla giriş edin

---

## SQL Server İstifadəçi İdarəetməsi
**SQL Server Management Studio (SSMS) ilə**

1. **SSMS-də qoşulun:**

- Server: localhost (lokal) və ya Tailscale IP
- Authentication: SQL Server Authentication
- Username: sa
- Password: docker-compose.yml-də təyin etdiyiniz şifrə


2. **New Query açın və aşağıdakı SQL-i çalışdırın:**

```bash
-- Yeni istifadəçi yaradın
CREATE LOGIN username WITH PASSWORD = 'YourStrong@Password123';
GO

-- Sysadmin icazəsi verin (tam icazə)
ALTER SERVER ROLE sysadmin ADD MEMBER username;
GO

-- Master database
USE master;
GO
CREATE USER username FOR LOGIN username;
GO

-- AdventureWorks2022 (əgər varsa)
USE AdventureWorks2022;
GO
CREATE USER usernmae FOR LOGIN username;
GO
ALTER ROLE db_owner ADD MEMBER username;
GO

-- İcazələri yoxlayın
SELECT 
    p.name AS username,
    r.name AS role_name
FROM sys.server_role_members srm
JOIN sys.server_principals p ON srm.member_principal_id = p.principal_id
JOIN sys.server_principals r ON srm.role_principal_id = r.principal_id
WHERE p.name = 'username';
GO
```

- **Terminaldan (sqlcmd ilə)**
```bash
sqlcmd quraşdırın
sudo apt install mssql-tools unixodbc-dev
``` 

## SQL Server-ə qoşulun
```bash
/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourStrong@Password123'
```

- **Yuxarıdakı SQL əmrlərini çalışdırın**

## Qoşulma və Test
**Eyni Şəbəkədən (Eyni Wi-Fi/LAN)**
**Qoşulma məlumatları:**

- Host: Ubuntu VM-in lokal IP-si (məsələn: 192.168.1.100)
- Port: 1433
- Username: username və ya sa
- Password: təyin etdiyiniz şifrə

**Fərqli Şəbəkədən (Tailscale ilə)**
**Qoşulma məlumatları:**

- Host: Tailscale IP (məsələn: 100.64.x.x)
- Port: 1433
- Username: username və ya sa
- Password: təyin etdiyiniz şifrə

**Test sorğuları**
```bash
-- Database-ləri siyahıya alın
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
```

---

## Mobil Cihazlardan Qoşulma (Android / iOS)

### SQL Server-ə mobil cihazlardan da qoşulmaq mümkündür. Bunun üçün Android və iOS üçün mövcud SQL tətbiqlərindən istifadə edə bilərsiniz.

**Tətbiq nümunələri:**
- Android: ```SQL Client```, ```DB Client```, ```SQLTool Pro```, ```RemoDB SQL Client MySQL, MsSQL```
- iOS: ```DB Client```, ```DBeaver Mobile```

**Qoşulma məlumatları:**

- Server: Tailscale IP (məs.: 100.64.x.x)

- Port: 1433

- Username: sa və ya yaratdığınız istifadəçi

- Password: şifrəniz

**Tətbiqdə New Connection → SQL Server seçərək bu məlumatlarla qoşula bilərsiniz.**

---

## Faydalı Əmrlər
### Docker Compose əmrləri 
- Konteyneri işə salmaq
```bash
sudo docker compose up -d
``` 

- Dayandırmaq
```bash
sudo docker compose down
```

- Yenidən başlatmaq
```bash
sudo docker compose restart
``` 

-  Statusu yoxlayın
```bash
sudo docker compose ps
```

- Logları görün
```bash
sudo docker compose logs -f sqlserver
```

- Container-ə daxil olun
```bash
sudo docker exec -it sqlserver /bin/bash
```

### SQL Server əmrləri (Container içində)
- **sqlcmd ilə qoşulun**
```bash
sudo docker exec -it sqlserver /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourStrong@Password123'
```

- **Database-ləri görün (birbaşa)**
```bash
sudo docker exec -it sqlserver /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourStrong@Password123' -Q "SELECT name FROM sys.databases;"
```

### Tailscale əmrləri
- Status yoxlayın
```bash
tailscale status
```

- IP ünvanını görün
```bash
tailscale ip -4
```

- Yenidən qoşulun
```bash
sudo tailscale up
```

- Çıxış edin
```bash
sudo tailscale down
```

### Backup və Restore
**Backup**

- Database backup
```bash
BACKUP DATABASE TestDB 
TO DISK = '/var/opt/mssql/data/TestDB.bak'
WITH FORMAT, COMPRESSION;
GO
```

- Backup faylını çıxarın
```bash
sudo docker cp sqlserver:/var/opt/mssql/data/TestDB.bak ~/TestDB.bak
```

### Restore
- Backup faylını konteynerə köçürün
```bash
sudo docker cp ~/TestDB.bak sqlserver:/var/opt/mssql/data/
```

- Database restore
```bash
RESTORE DATABASE TestDB
FROM DISK = '/var/opt/mssql/data/TestDB.bak'
WITH REPLACE;
GO
```

---

## Firewall Konfiqurasiyası (Lazım olsa)
- UFW ilə port 1433-ü açın
```bash
sudo ufw allow 1433/tcp
sudo ufw reload
```
- UFW statusunu yoxlayın
```bash
sudo ufw status
```

---

# Problemlərin Həlli
## Container işləmir
 
- Logları yoxlayın
```bash
sudo docker compose logs sqlserver
```

- Yenidən başladın
```bash
sudo docker compose restart
```
## Qoşulma problemi

1. **Firewall yoxlayın:**

```bash   
sudo ufw status
```

2. **SQL Server hazırdırmı:**

```bash   
sudo docker compose logs -f sqlserver | grep "ready"
```

3. **Port açıqdırmı:**

```bash   
sudo netstat -tulpn | grep 1433
```

## Şifrə dəyişmək
```bash
ALTER LOGIN sa WITH PASSWORD = 'NewStrong@Password456';
GO
```

---

## Təhlükəsizlik Tövsiyələri

✅ **Güclü şifrələr istifadə edin (minimum 8 simvol, böyük/kiçik hərf, rəqəm, simvol)**

✅ **sa istifadəçisi əvəzinə fərqli istifadəçilər yaradın**

✅ **Firewall konfiqurasiyasını düzgün təyin edin**

✅ **Mütəmadi backup alın**

✅ **SQL Server-i güncel saxlayın**

✅ **Tailscale istifadə edərək internetdən birbaşa port açmayın**

---

## Əlavə Resurslar

- [Linux Ubuntu Desktop](https://ubuntu.com/desktop)

- [Linux Ubuntu Server](https://ubuntu.com/server)

- [Docker Documentation](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

- [SQL Server on Linux](https://hub.docker.com/r/microsoft/mssql-server)

- [Tailscale Documentation](https://tailscale.com/kb/1017/install)

- [SQL Server Best Practices](https://learn.microsoft.com/en-us/sql/relational-databases/security/security-center-for-sql-server-database-engine-and-azure-sql-database?view=sql-server-ver17)


---

**Xəyyam Əhmədov [Linkedin](https://www.linkedin.com/in/x%C9%99yyam-%C9%99hm%C9%99dov)**
