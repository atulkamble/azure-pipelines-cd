// Nginx - Webserver on AzureVM

sudo apt update -y
sudo apt install nginx 
sudo systemctl enable nginx 
sudo systemctl start nginx 
sudo rm index.nginx-debian.html
echo "<h1>webserver</h1>" > /var/www/html/index.html 

http://20.151.97.91/

// Deployment on ACR
// keep docker desktop running 

az login

az acr login --name atulkamble

git clone https://github.com/atulkamble/FlaskApp-ACR-ACI.git 
cd FlaskApp-ACR-ACI

python -- version
pip --version
pip install -r requirements.txt 
python app.py 

http://localhost:5000

5IuNcPiQNzxqXGmVkMyDJvQcNZcJ8dTxw2Q8SKTXf5kmoLb9MrvNJQQJ99CIACBsN54Eqg7NAAACAZCRDnEp

docker buildx build --platform linux/amd64,linux/arm64 -t atulkamble.azurecr.io/cloudnautic/pythonapp:latest --load .
docker images
docker push atulkamble.azurecr.io/cloudnautic/pythonapp
docker pull atulkamble.azurecr.io/cloudnautic/pythonapp
docker run -d -p 5000:5000 atulkamble.azurecr.io/cloudnautic/pythonapp

// copy azure-pipelines.yml from https://github.com/atulkamble/FlaskApp-ACR-ACI

// Tip: Service Connection - 

do not select ARM (Azure Resource Manager)

Docker Registry - ACR - Workload Identity Federation (Automatic)\


document this 



// Nginx - Webserver on AzureVM

sudo apt update -y
sudo apt install nginx 
sudo systemctl enable nginx 
sudo systemctl start nginx 
sudo rm index.nginx-debian.html
echo "<h1>webserver</h1>" > /var/www/html/index.html 

http://20.151.97.91/














