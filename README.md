# Ecole42Paris_BADASS
Using ubuntu LTS 24.04.2 in Virtual box


[gns3 installation](https://docs.gns3.com/docs/getting-started/installation/linux/) \
[docker installation](https://docs.docker.com/engine/install/ubuntu/)

When gns3 get a problem with root permission with docker, use this command 
```sh
sudo usermod -aG docker $USER
```
and reboot or logout/login the session,
```sh
groups
```
you must have docker in the groups.

---
how build the dockerfile to image.
```sh
sudo docker build -t [container name] -f [dockerfile] .
```