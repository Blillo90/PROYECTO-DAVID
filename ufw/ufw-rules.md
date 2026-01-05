sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22
sudo ufw allow from 192.168.56.0/24
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable