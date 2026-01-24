
### Docker lab 

#### url 
```bash 
http://172.16.116.182:3000/
```
#### username 
```bash 
dev
```
#### pass 
```bash 
UBMongolia1234
```

### Acer server 

#### ip 
```bash 
192.168.81.163
```

#### clickhouse url 
```bash 
http://192.168.81.163:8123/play
```


### Docker hub Aid

#### username 
```bash 
aidgrapecity
```

#### password
```
UBMongolia1234
```

### push oracle image to docker hub

```bash 
docker login -u aidgrapecity
```

```bash 
docker tag oracle/database:19.3.0-ee aidgrapecity/oracle-database:19.3.0-ee
```

```bash 
docker push aidgrapecity/oracle-database:19.3.0-ee
```
