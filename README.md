# Roadmap.sh-005

# Setup Prometheus and Grafana for remote server using docker

Requirements:
 - An Ubuntu server reachable on the internet with port 22(ssh) open and docker installed
 - Local machine with docker installed


# Setup Remote
### Install nginx, nginx-prometheus-exporter & prometheus/node-exporter

SSH to your remote server and follow the steps:

```
git clone https://github.com/jammie-jelly/Roadmap.sh-005
```

Then do 
```
cd server/prom-graf && docker compose up -d
```

This will run the services below:

```
services:
  nginx:
    image: nginx
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro

  nginx_exporter:
    image: nginx/nginx-prometheus-exporter:latest
    container_name: nginx_exporter
    command:
      - --nginx.scrape-uri=http://nginx:80/stub_status
    ports:
      - "9113:9113"
    depends_on:
      - nginx

  node_exporter:
    image: quay.io/prometheus/node-exporter:latest
    container_name: node_exporter
    command:
      - '--path.rootfs=/host'
    network_mode: host
    pid: host
    restart: unless-stopped
    volumes:
      - '/:/host:ro,rslave'
```


# On your local machine
### Let's setup prometheus to connect to our services and allow grafana to visualize the stats

Fetch the sources once more

```
git clone https://github.com/jammie-jelly/Roadmap.sh-005
```

Then take a close look at the compose file in the parent directory. You will be customizing 2 variables to your specifications:
  - the ssh private key you use to connect to the remote server
  - the remote server's ssh login details i.e username@ip

```
    volumes:
      - ~/.ssh/gg.pem:/root/.ssh/id_rsa:ro
      
    command: >
          /bin/sh -c "apk add --no-cache openssh && ssh -o StrictHostKeyChecking=no -N \
          -L 0.0.0.0:9113:localhost:9113 \
          -L 0.0.0.0:9100:localhost:9100 \
          ubuntu@13.53.102.210 -i /root/.ssh/id_rsa"
```

Once you have filled in these details. Do `docker compose up -d` for the compose file to get prometheus, ssh tunnel and grafana running.

```
services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    ports:
      - "9090:9090" 
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro  
    networks:
      - mynetwork
      
  ssh-tunnel:
    image: alpine
    container_name: ssh-tunnel
    ports:
      - "9113:9113"
      - "9100:9100"
    command: >
      /bin/sh -c "apk add --no-cache openssh && ssh -o StrictHostKeyChecking=no -N \
      -L 0.0.0.0:9113:localhost:9113 \
      -L 0.0.0.0:9100:localhost:9100 \
      ubuntu@13.53.102.210 -i /root/.ssh/id_rsa"
    volumes:
      - ~/.ssh/gg.pem:/root/.ssh/id_rsa:ro
    networks:
      - mynetwork
    depends_on:
      - prometheus

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"  
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin  
    depends_on:
      - prometheus
    networks:
      - mynetwork
    volumes:
      - grafana_data:/var/lib/grafana  

networks:
  mynetwork:
    driver: bridge

volumes:
  grafana_data:
```

On your browser go to the grafana endpoint `http://localhost:3000` you will be greeted with a login page. The default password for user admin is `admin` if you did not change `GF_SECURITY_ADMIN_PASSWORD`.

Press skip password change option and go to status > targets on prometheus to confirm the services are reachable on
`http://localhost:9090/targets?search=`. 

You should see status `UP` in green for the 2 services as defined in `prometheus.yml`.

![image](https://github.com/user-attachments/assets/bee38090-38db-4050-80ef-7eb46356df30)


If your services are green, click on `add your first data source`, click on `prometheus` then add `http://prometheus:9090` as the Prometheus server URL.

Press `save & test` then in the feedback popup click on `build dashboard` option or in the menu.

You can add a dashboard and customize it or you can import a custom [pre-configured dashboard](https://grafana.com/grafana/dashboards).

Click on `Import a dashboard`, then add and `load` ids:

 - `1860` for Node Exporter Full
 - `11199` for NGINX by nginxinc


# The results:


![Screenshot From 2024-11-07 18-39-34](https://github.com/user-attachments/assets/ef687a9c-ade0-4600-a6cb-575c1f9a8d2d)

![Screenshot From 2024-11-07 18-39-09](https://github.com/user-attachments/assets/267abea5-5b33-45e2-a230-18ad047b72eb)





Part of this challenge: https://roadmap.sh/projects/monitoring
