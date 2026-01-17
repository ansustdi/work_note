



---

## 🔐 DockerHub Aid
> [!key] DockerHub Credentials

| Field | Value |
|---|---|
| **Username** | `aidgrapecity` |
| **Password** | `UBMongolia1234` |

---

## 🛠️ Docker Commands
> [!terminal] Build · Tag · Push · Export · Import

### 🔨 Build Image
```bash
docker build -t data-pipeline-tools:5.0 .
```

### 🏷️ Tag Image
```bash
docker tag a536a88154fc aidgrapecity/data-pipeline-tools:5.0
```

### 🚀 Push Image
```bash
docker push aidgrapecity/data-pipeline-tools:5.0
```

### 📤 Export Image
```bash
docker export data-pipeline-tools -o data-pipeline-tools.tar
```

### 📥 Import Image
```bash
docker import data-pipeline-tools.tar data-pipeline-tools:1.0
```

---

## 🧹 Docker Compose Temp File Cleanup
> [!trash] Temporary Files

```bash
rm /var/tmp/docker-compose.yaml.swp
```

---



## 🏷️ Tags
#docker #dockerhub #devops #containers
