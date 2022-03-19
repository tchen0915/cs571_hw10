# Week 10 Homework Signature Project: MongoDB + Python Flask Web Framework + REST API + GKE

- create instance on GCP and add external IP
- login VM instance

```bash
gcloud auth login
```

# Step1. Create MongoDB using Persistent Volume on GKE

1. Create a cluster

```bash
sudo apt-get install kubectl
gcloud container clusters create kubia --num-nodes 1 --machine-type e2-micro \
--zone us-west4-b
```

1. Create persistent volume

```bash
gcloud compute disks create --size=10GiB --zone=us-west4-b mongodb
```

1. Create a MongoDB deployment
- content

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
spec:
  selector:
    matchLabels:
     app: mongodb
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - image: mongo
          name: mongo
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongodb-data
              mountPath: /data/db
      volumes:
        - name: mongodb-data
          gcePersistentDisk:
            pdName: mongodb
            fsType: ext4
```

- command

```bash
sudo vi mongodb-deployment.yaml
kubectl create -f mongodb-deplyment.yaml
kubectl get pods
```

![Untitled](Week%2010%20Ho%20838bb/Untitled.png)

1. Create a MongoDB service
- content

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  type: LoadBalancer
  ports:
    - port: 27017
      targetPort: 27017
  selector:
    app: mongodb
```

- command

```bash
sudo vi mongodb-service.yaml
kubectl create -f mongodb-service.yaml
kubectl get svc
```

![Untitled](Week%2010%20Ho%20838bb/Untitled%201.png)

1. Test MongoDB connection in the deployment using External IP

```bash
kubectl exec -it mongodb-deployment-57dc68b4bd-bhvv8 -- bash
mongo 34.125.211.195
```

![Untitled](Week%2010%20Ho%20838bb/Untitled%202.png)

1. Insert some records in MongoDB

```bash
use studentdb
db.students.insert({ student_id: 11111, student_name: "Bruce Lee", grade: 84})
db.students.insert({ student_id: 22222, student_name: "Jackie Chen", grade: 93 })
db.students.insert({ student_id: 33333, student_name: "Jet Li", grade: 88})
db.students.find()
```

![Untitled](Week%2010%20Ho%20838bb/Untitled%203.png)

# Step2. Modify studentServer to get records from MongoDB and deploy to GKE

1. Create JS file
- content

```jsx
var http = require('http');
var url = require('url');
var mongodb = require('mongodb');
const {
  MONGO_URL,
  MONGO_DATABASE
} = process.env;

var MongoClient = mongodb.MongoClient;
var uri = `mongodb://${MONGO_URL}/${MONGO_DATABASE}`;
console.log(uri);

var server = http.createServer(function (req, res) {
  var result;
  // req.url = /api/score?student_id=11111
  var parsedUrl = url.parse(req.url, true);
  var student_id = parseInt(parsedUrl.query.student_id);
  // match req.url with the string /api/score
  if (/^\/api\/score/.test(req.url)) {
    // e.g., of student_id 1111
    MongoClient.connect(uri,{ useNewUrlParser: true, useUnifiedTopology:
    true }, function(err, client){
      if (err)
        throw err;
      var db = client.db("studentdb");
      db.collection("students").findOne({"student_id":student_id},
      (err, student) => {
        if(err)
          throw new Error(err.message, null);
        if (student) {
          res.writeHead(200, { 'Content-Type': 'application/json'
        })
          res.end(JSON.stringify(student)+ '\n')
        } else {
          res.writeHead(404);
          res.end("Student Not Found \n");
        }
      });
    });
  } else {
    res.writeHead(404);
    res.end("Wrong url, please try again\n");
  }
});
server.listen(8080);
```

- command

```bash
sudo vi studentServer.js
```

1. Create Dockerfile
- content

```docker
FROM node:14
ADD studentServer.js /studentServer.js
RUN npm install mongodb
ENTRYPOINT ["node", "studentServer.js"]
```

- command

```bash
sudo vi Dockerfile
```

1. Build the student server docker image

```bash
sudo docker build -t tchen0915/studentserver .
```

![Untitled](Week%2010%20Ho%20838bb/Untitled%204.png)

1. Push to DockerHub

```bash
sudo docker login
sudo docker push tchen0915/studentserver
```

![Untitled](Week%2010%20Ho%20838bb/Untitled%205.png)

# Step 3. Create a python Flask bookshelf REST API server and deploy to GKE

1. Create python file
- content

```python
from flask import Flask, request, jsonify
from flask_pymongo import PyMongo
from flask import request
from bson.objectid import ObjectId
import socket
import os

app = Flask(__name__)
app.config["MONGO_URI"] = "mongodb://"+os.getenv("MONGO_URL")+"/"+os.getenv("MONGO_DATABASE")
app.config['JSONIFY_PRETTYPRINT_REGULAR'] = True
mongo = PyMongo(app)
db = mongo.db

@app.route("/")
def index():
  hostname = socket.gethostname()
  return jsonify(
    message="Welcome to bookshelf app! I am running inside {} pod!".format(hostname)
  )

@app.route("/books")
def get_all_tasks():
  books = db.bookshelf.find()
  data = []
  for book in books:
    data.append({
    "id": str(book["_id"]),
    "Book Name": book["book_name"],
    "Book Author": book["book_author"],
    "ISBN" : book["ISBN"]
    })
  return jsonify(data)

@app.route("/book", methods=["POST"])
def add_book():
  book = request.get_json(force=True)
  db.bookshelf.insert_one({
    "book_name": book["book_name"],
    "book_author": book["book_author"],
    "ISBN": book["isbn"]
  })
  return jsonify(message="Task saved successfully!")

@app.route("/book/<id>", methods=["PUT"])
def update_book(id):
  data = request.get_json(force=True)
  print(data)
  response = db.bookshelf.update_many({"_id": ObjectId(id)}, {"$set":
  {"book_name": data['book_name'],
  "book_author": data["book_author"], "ISBN": data["isbn"]
  }})
  if response.matched_count:
    message = "Task updated successfully!"
  else:
    message = "No book found!"
  return jsonify(message=message)

@app.route("/book/<id>", methods=["DELETE"])
def delete_task(id):
  response = db.bookshelf.delete_one({"_id": ObjectId(id)})
  if response.deleted_count:
    message = "Task deleted successfully!"
  else:
    message = "No book found!"
  return jsonify(message=message)

@app.route("/tasks/delete", methods=["POST"])
def delete_all_tasks():
  db.bookshelf.remove()
  return jsonify(message="All Books deleted!")

if __name__ == "__main__":
  app.run(host="0.0.0.0", port=5000)
```

- command

```bash
sudo vi bookshelf.py
```

1. Create Dockerfile
- content

```docker
FROM python:alpine3.7
ADD bookshelf.py /bookshelf.py
RUN pip3 install flask flask_pymongo
ENV PORT 5000
EXPOSE 5000
ENTRYPOINT ["python3"]
cmd [ "bookshelf.py" ]
```

- command

```bash
sudo vi bookshelf.Dockerfile
```

1. Build bookshelf app into docker image and push to DockerHub

```bash
sudo docker build -f bookshelf.Dockerfile -t tchen0915/bookshelf .
sudo docker push tchen0915/bookshelf
```

![Untitled](Week%2010%20Ho%20838bb/Untitled%206.png)

# Step 4. Create configMap for both app to store MongoDB URL and MongoDB name

1. Create configMap file
- content

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: studentserver-config
data:
  MONGO_URL: 34.125.211.195
  MONGO_DATABASE: mydb
```

- command

```bash
sudo vi studentserver-configmap.yaml
kubectl create -f studentserver-configmap.yaml
```

1. Create configMap file for bookshelf app
- content

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: bookshelf-config
data:
  MONGO_URL: 34.125.211.195
  MONGO_DATABASE: mydb
```

- command

```bash
sudo vi bookshelf-configmap.yaml
kubectl create -f bookshelf-configmap.yaml
```

![Untitled](Week%2010%20Ho%20838bb/Untitled%207.png)

# Step 5. Expose 2 apps using ingress with Nginx, so we can put them on the same domain but different PATH

1. Create student server deployment
- content

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: studentserver-deployment
  labels:
    app: studentserver-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
     app: studentserver-deployment
  template:
    metadata:
      labels:
        app: studentserver-deployment
    spec:
      containers:
        - image: tchen0915/studentserver
          imagePullPolicy: Always
          name: studentserver-deployment
          ports:
            - containerPort: 8080
          env:
            - name: MONGO_URL
              valueFrom:
                configMapKeyRef:
                  name: studentserver-config
                  key: MONGO_URL
            - name: MONGO_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: studentserver-config
                  key: MONGO_DATABASE
```

- command

```bash
sudo vi studentserver-deployment.yaml
kubectl create -f studentserver-deployment.yaml
```

1. Create student server service
- content

```yaml
apiVersion: v1
kind: Service
metadata:
  name: studentserver-service
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: studentserver-deployment
```

- command

```bash
sudo vi studentserver-service.yaml
kubectl create -f studentserver-service.yaml
```

1. Create bookshelf deployment
- content

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookshelf-deployment
  labels:
    app: bookshelf-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
     app: bookshelf-deployment
  template:
    metadata:
      labels:
        app: bookshelf-deployment
    spec:
      containers:
        - image: tchen0915/bookshelf
          imagePullPolicy: Always
          name: bookshelf-deployment
          ports:
            - containerPort: 5000
          env:
            - name: MONGO_URL
              valueFrom:
                configMapKeyRef:
                  name: bookshelf-config
                  key: MONGO_URL
            - name: MONGO_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: bookshelf-config
                  key: MONGO_DATABASE
```

- command

```bash
sudo vi bookshelf-deployment.yaml
kubectl create -f bookshelf-deployment.yaml
```

1. Create bookshelf service
- content

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bookshelf-service
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 5000
  selector:
    app: bookshelf-deployment
```

- command

```bash
sudo vi bookshelf-service.yaml
kubectl create -f bookshelf-service.yaml
```

![Untitled](Week%2010%20Ho%20838bb/Untitled%208.png)

![Untitled](Week%2010%20Ho%20838bb/Untitled%209.png)

1. Check status

```bash
kubectl get deploy
kubectl get po
kubectl get svc
```

![Untitled](Week%2010%20Ho%20838bb/Untitled%2010.png)

1. Test student server and bookshelf with IP

```bash
curl 34.125.123.39:8080/api/score?student_id=33333
curl 34.125.161.167:5000/books
```

![Untitled](Week%2010%20Ho%20838bb/Untitled%2011.png)

# Step 6. Exposing multiple services

1. Create ingress file
- content

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubia-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /studentserver
        pathType: Prefix
        backend:
          service:
            name: studentserver-service
            port:
              number: 80
      - path: /bookshelf
        pathType: Prefix
        backend:
          service:
            name: bookshelf-service
            port:
              number: 80
```

- command

```bash
sudo vi ingress.yaml
kubectl create -f ingress.yaml
kubectl get ingress
```

![Untitled](Week%2010%20Ho%20838bb/Untitled%2012.png)

![Untitled](Week%2010%20Ho%20838bb/Untitled%2013.png)

1. Add line to the bottom of the /etc/hosts file

```bash
sudo vi /etc/hosts
```

content of /etc/hosts file

```bash
.....
34.111.85.52 kubia.example.com
```

# Step 7. Test Endpoint

1. Test student server

```bash
curl http://kubia.example.com/studentserver/api/score?student_id=33333
```

![Untitled](Week%2010%20Ho%20838bb/Untitled%2014.png)

1. Test bookshelf

```bash
curl http://kubia.example.com/studentserver/books
```

```bash
Add a book
curl -X POST -d "{\"book_name\": \"cloud computing\",\"book_author\":
\"unkown\", \"isbn\": \"123456\" }" http://cs571.project.com/bookshelf/book
```

```bash
Update a book
curl -X PUT -d "{\"book_name\": \"123\",\"book_author\": \"test\", \"isbn\":
\"123updated\" }" http://cs571.project.com/bookshelf/book/id
```

```bash
Delete a book
curl -X DELETE cs571.project.com/bookshelf/book/id
```