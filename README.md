# Web API workshop

This workshop is targeted for students of EPITA SIGL 2022.

You will learn how to create a Web API using NodeJS and Express.

This template API will expose a single route to get resources for Garlaxy.

## Step 1: Create our web API on localhost

**Objective**: Expose your "ressource" catalog on /v1/resource?page=1&limit=10

### Create a new NodeJS project

From your group's repository (ex: groupe13): 
- create a backend folder (same level as frontend)
```sh
# from your project's repository, where you can see your frontend/ folder
mkdir -p backend
```
- initiate a new nodejs project using node v16
```sh
cd backend
nvm use v16
npm init -y
```
- add express dependency
```sh
# still from backend/ folder
npm i --save express
```
- add a first 'Hello World' route to `backend/src/server.js`
```sh
# From backend/; create a new src/ folder and server.js file
mkdir -p src
touch src/server.js
```
```js
// From backend/src/server.js
const express = require("express");
const app = express();
const port = 3030;

app.get("/", (req, res) => {
  res.send("Hello World!");
});

app.listen(port, () => {
  console.log(`Listening at http://localhost:${port}`);
});
```

Run the hello world code from your terminal:
```sh
# from backend/
# make sure you are on correct version of node: nvm use v16
node src/server.js
```

Your web api should listen on [localhost:3030](http://localhost:3030/).
If you go from your browser, you should see `Hello World!`.

### Code your web API

Now, let's create our database-less resource web API.

Create and copy the content of the following file in your `backend/` folder:
- [backend/.gitignore](https://github.com/arla-sigl-2022/groupe-13/blob/b5de4851d7d612327421790a84b8eae00c760f29/backend/.gitignore)    ignore trash files when using git add
- [backend/src/server.js](https://github.com/arla-sigl-2022/groupe-13/blob/b5de4851d7d612327421790a84b8eae00c760f29/backend/src/server.js)  where you expose your route
- [backend/src/database.js](https://github.com/arla-sigl-2022/groupe-13/blob/b5de4851d7d612327421790a84b8eae00c760f29/backend/src/database.js)  where we set our data. This is NOT a real database, we just keep default data in memory. Restarting the app will not persist any data.
- [backend/src/utils.js](https://github.com/arla-sigl-2022/groupe-13/blob/b5de4851d7d612327421790a84b8eae00c760f29/backend/src/utils.js)    Utilitary functions to help you recover query string parameters (e.g. `page` and `limit` from resource route)

Now run again your web API locally:
```sh
# make sure you've quit previous node src/server.js process
# from backend/
nvm use v16
node src/server.js
```
You should be able to query data from [localhost:3030/v1/resource?page=1&limit=5](http://localhost:3030/v1/resource?page=1&limit=5).
You can query those data via your browser directly, or via [cURL](https://curl.se/) from your terminal:

```sh
# you need to escape special characters
curl http://localhost:3030/v1/resource\?page\=1\&limit\=5
# or put host in strings
curl 'http://localhost:3030/v1/resource?page=1&limit=5'
# Response:
# {"resources":[{"id":1,"planete":"Marlax","ressource":"Arlaminium","prix":0.73},
# {"id":2,"planete":"Sarlax","ressource":"Arlium","prix":3.25},
# {"id":3,"planete":"Larlune","ressource":"Rarladium","prix":1.42},
# {"id":4,"planete":"Ozinia","ressource":"Elekectium","prix":1.2},
# {"id":5,"planete":"Pinziamia","ressource":"Nontcite","prix":1}]}
```

> Note: feel free to check [groupe13's STEP 1](https://https://github.com/arla-sigl-2022/groupe-13/tree/b5de4851d7d612327421790a84b8eae00c760f29/backend)

## Step 2: Deploy your web API

**Objective**: have your "ressource" web API deployed on https://api.groupeXX.arla-sigl.fr (groupXX behing replaced by your groupe)
### Create a Dockerfile for your NodeJS web API

- Create a new Dockerfile from `backend/` folder containing:
```dockerfile
# From backend/Dockerfile
FROM node:16

COPY . /code
WORKDIR /code

RUN npm i
CMD ["node", "src/server.js"]

EXPOSE 3030
```

Try out locally using `--init` flag. 

> Note: you need to run your docker image with `—init` flag because of [this known issue from docker + node together](https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md#handling-kernel-signals)
```sh
# from backend
docker build -t garlaxy-web-api:local .
#...
docker run --init -p 3033:3030 garlaxy-web-api:local
```
Now make sure you can get "ressource" data on [localhost:3033/v1/resource?page=1&limit=5](http://localhost:3033/v1/resource?page=1&limit=5)

### Create a new job in your github workflow

Now that you have a runnning docker setup for your node web API, let's add a new job to your current CD workflow.

From `.github/workflows/main.yml`, making sure to replace `groupe-XX` by your group number 
and `arla-sigl-2022` by your organization/user's name on github:
```yaml
# inside .github/workflows/main.yml
# ....
jobs:
  build-frontend:
    # ... 
  build-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: build and publish backend docker image
        working-directory: backend
        run: |
          docker build -t docker.pkg.github.com/arla-sigl-2022/groupe-XX/garlaxy-web-api:${{ github.sha }} .
          docker login docker.pkg.github.com -u ${{ secrets.DOCKER_USER }} -p ${{ secrets.DOCKER_PASSWORD }}
          docker push docker.pkg.github.com/arla-sigl-2022/groupe-XX/garlaxy-web-api:${{ github.sha }} 
      - name: Deploy on production VM
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            docker login docker.pkg.github.com -u ${{ secrets.DOCKER_USER }} -p ${{ secrets.DOCKER_PASSWORD }}
            docker pull docker.pkg.github.com/arla-sigl-2022/groupe-XX/garlaxy-web-api:${{ github.sha }}
            (docker stop garlaxy-web-api && docker rm garlaxy-web-api) || echo "Nothing to stop"
            docker run -d --name garlaxy-web-api --init --network web --label traefik.enable=true --label traefik.docker.network=web --label traefik.frontend.rule=Host:api.groupeXX.arla-sigl.fr --label traefik.frontend.port=3030 docker.pkg.github.com/arla-sigl-2022/groupe-XX/garlaxy-web-api:${{ github.sha }}
```

It's very similar to your frontend build job, expect:
- `working-directory` is `backend`
- `image-name` and `container-name` are `garlaxy-web-api`
- `label` in traefik is `api.groupeXX.arla-sigl.fr`
- `--init` flag should be there when running the docker container

Commit/push your changes, and you should trigger CD workflow.
After few minutes, you should be able to access your web API on https://api.groupeXX.arla-sigl.fr 

> Note: You can check out [groupe 13's Step 1 to Step 2 changes](https://github.com/arla-sigl-2022/groupe-13/pull/2/commits/eb40c9978799d0d46c95a60b24b12abb17410a46)
## Step 3: Integrate web API to Garlaxy frontend

**Objective**: Inside your frontend, fetch the list of "ressource" served by your new web API.

### Make it work on localhost

Make sure to integrate the following changes to your `frontend/`:
- [frontend/src/Content.js](https://github.com/arla-sigl-2022/groupe-13/pull/2/commits/a8d36181617eef3d5d5dad6e1433bba35007ac69#diff-6ee7f9aad46ed7e9368b1c441b405d6ba99294a53013fea551657d30943ea3ad)
- [frontend/src/GarlaxyContext.js](https://github.com/arla-sigl-2022/groupe-13/pull/2/commits/a8d36181617eef3d5d5dad6e1433bba35007ac69#diff-1ed3a915763da17e7953989df88a229bb60d7cd3a88403b2c56d4122a0b2b599)

Now, run both your `frontend/` and `backend/` from two different terminal sessions:

```sh
# from frontend/
nvm use v16
npm start
```

```sh
# from backend/
nvm use v16
node src/server.js
```

You should see your "Ressource" table with 10 entries coming from your web API on [localhost:3000](http://localhost:3000).

### Make it work on production

Now, let's make sure you can access the web API not from localhost:3030 but from your api.groupeXX.arla-sigl.fr domain.

If you try to just replace the "http://localhost:3030" by your already deployed web API from Step 1, you'll encounter an issue with CORS.

> Note: you can check out this [Very nice article about CORS on MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)

To solve this issue of cross-origin domains ,
you will install a new Express middleware called [cors](https://www.npmjs.com/package/cors).

- [cors middleware], implements mandatory Cross-Origin Ressource Sharing check triggerd by browsers. This middleware is necessary if your frontend is requesting an API on a different domain than the API domain (e.g. from localhost:3030 to api.groupeXX.arla-sigl.fr and groupeXX.arla-sigl.fr to api.groupeXX.arla-sigl.fr)

From your `backend/`, add the cors middleware:
```sh
# from backend/
nvm use v16
npm install --save cors
```
And, apply the following changes to both frontend and backend:
- [backend/src/server.js](https://github.com/arla-sigl-2022/groupe-13/commit/d40b36cddffd702a819480ab2dd3b7d96efac61d#diff-36e2c2dd1e67a7419cef780285f514e743e48ac994a01526288acd31707e09ae)
- [frontend/src/Content.js](https://github.com/arla-sigl-2022/groupe-13/commit/d40b36cddffd702a819480ab2dd3b7d96efac61d#diff-6ee7f9aad46ed7e9368b1c441b405d6ba99294a53013fea551657d30943ea3ad)

Commit and push your changes, and you should be able to use the new version of Garlaxy that uses your own created web API!

