# Redash
## Tutorial setup Redash di Docker dengan OS Linux Debian 12 melalui Oracle VirtualBox 

**Disclaimer**
> Tutorial ini dibuat berdasarkan pengalaman pribadi saat melakukan proses instalasi. Tidak menutup kemungkinan terdapat kesamaan alur atau struktur dengan panduan lain yang telah dipublikasikan sebelumnya. Apabila terdapat kemiripan, khususnya yang berkaitan dengan hak cipta, hal tersebut sama sekali tidak disengaja. Saya tidak merujuk atau menyalin sumber manapun dalam penyusunan konten ini. Seluruh materi disusun dengan niat baik untuk keperluan pembelajaran dan berbagi pengetahuan, agar dapat memberikan manfaat secara luas bagi komunitas.

**Tingkat Kesulitan**: *Menengah hingga Lanjutan*
> Panduan ini ditujukan bagi individu yang telah memiliki pemahaman dasar hingga menengah dalam bidang teknologi informasi, khususnya terkait proses instalasi perangkat lunak, penggunaan terminal Linux, serta pemahaman terhadap konsep dan prosedur teknis yang umum dalam dunia IT.

### Bahan-bahan yang diperlukan
1. Oracle VirtualBox (https://www.virtualbox.org/wiki/Downloads)
2. Distro Linux Debian 12.11.10 (https://www.debian.org/distrib/)

### INSTALASI VM
1. Install Oracle VirtualBox
2. Spesifikasi yang saya gunakan:
   | <!-- -->    | <!-- -->    |
   | -------- | ------- |
   | OS | debian-12.11.10-amd64-netinst.iso |
   | Base Memory | 8192MB |
   | Processor | 2 CPU |
   | HDD | 20GB |
   | <!-- -->    | <!-- -->    |
4. Install Debian 12, saya memilih installasi minimalis, hanya menggunakan `standard system utilities`
5. Setelah berhasil install Debian 12, power off VMnya terlebih dahulu
6. Set Network nya:
	* Adapter 1: `NAT`
	* Adapter 2: `Host-only Adapter`
7. Start VMnya

### SETTING JARINGAN
1. Login dengan menggunakan akses `root`
2.
    ```
    apt update
    ```
3.
    ```
    ip a
    ```
    pastikan jaringan *host-only adapter* (`enp0s8`) ada
4.
   ```
   nano /etc/network/interfaces.d/enp0s8
   ```
6. masukan ini:
   ```
   auto enp0s8
   iface enp0s8 inet static
     address 192.168.56.101
     netmask 255.255.255.0
   ```
   kemudian tekan `ctrl + o` untuk save, kemudian `ctrl + x` untuk tutup nano
7.
   ```
   systemctl restart networking
   ```
   untuk restart networking
9. cek kembali di 
   ```
   ip a
   ```
   apakah ip address untuk local sudah benar contoh disini saya menggunakan ipaddress statis di `192.168.56.101`
10. test ping ke ip host dan sebaliknya. pastikan dapat terhubung kedua jaringan tersebut

### Install Docker
1. ```
    apt -y install ca-certificates curl gnupg lsb-release git
    ```
2. 
    ```
    install -m 0755 -d /etc/apt/keyrings
    ```
3. 
    ```
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    ```
4. 
    ```
    chmod a+r /etc/apt/keyrings/docker.gpg
    ```
5. 
    ```
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```
6. 
    ```
    apt update
    ```
7. 
    ```
    apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```
8. 
    ```
    usermod -aG docker $USER
    ```
9. 
    ```
    newgrp docker
    ```

### Install Portainer
1. 
   ```
   docker volume create portainer_data
   ```
2.
   ```
   docker run -d -p 8000:8000 -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
   ```
4. 
   ```
   docker start portainer
   ```
3. Buka browser di host (PC) ketik `https://192.168.56.101:9443`, jika benar, maka akan keluar settingan awal dari portainer
4. buat username dan password untuk admin portainer

### Install Redash
1. Buat Environment terlebih dahulu dengan menggunakan wizard
2. pilih `docker standalone` > start wizard
3. pilih `socket` > isikan nama environmentnya, saya menggunakan `env-docker` > klik `connect`
4. klik home di menu sebelah kiri, pastikan `env-docker` sudah ada
5. klik `env-docker` > `stacks` > `add stacks`
6. copas ini ke dalam field `web editor`
```
version: "3"
services:
  postgres:
    image: postgres:12-alpine
    restart: always
    environment:
      POSTGRES_PASSWORD: redash
      POSTGRES_USER: redash
      POSTGRES_DB: redash
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - redash-net

  redis:
    image: redis:7-alpine
    restart: always
    networks:
      - redash-net

  redash:
    image: redash/redash:latest
    depends_on:
      - postgres
      - redis
    environment:
      REDASH_DATABASE_URL: "postgresql://redash:redash@postgres/redash"
      REDASH_REDIS_URL: "redis://redis:6379/0"
      REDASH_SECRET_KEY: "hogehoge"
      REDASH_COOKIE_SECRET: "mogemoge"
      REDASH_WEB_WORKERS: 2
    ports:
      - "5000:5000"
    restart: always
    networks:
      - redash-net

  worker:
    image: redash/redash:latest
    depends_on:
      - postgres
      - redis
    environment:
      REDASH_DATABASE_URL: "postgresql://redash:redash@postgres/redash"
      REDASH_REDIS_URL: "redis://redis:6379/0"
      REDASH_SECRET_KEY: "hogehoge"
      REDASH_COOKIE_SECRET: "mogemoge"
    command: worker
    restart: always
    networks:
      - redash-net

volumes:
  postgres-data:

networks:
  redash-net:
    external: true
```
