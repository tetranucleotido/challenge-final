name: App Final

#pipeline = develop
#pipeline2= testing

on:
  push:
    branches: 
      - develop      
  pull_request:
    branches:
      - main
      - testing

env:
  REGISTRY: tetranucleotido
  REPOSITORY1: frontend
  REPOSITORY2: products
  REPOSITORY3: shopping

jobs:

  Init:
    name: 📍 Installs Dependencies
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-node@v3
      with:
        node-version: '14'
    - run: npm install

  Test:
    name: 🕯 Unit Test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3  
    - uses: actions/setup-node@v3
      with:
        node-version: '14'
    - run: npm run test
    
  Build:
    name: 📌 Build and Push DockerHub"
    runs-on: ubuntu-latest
    needs: [Init,Test]
    steps:
    - uses: actions/checkout@v3
    - name: Set up QEMU
      uses: docker/setup-qemu-action@v2
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    - name: Build and push
      run: |
        VERSION1=`jq -r '.version' frontend/package.json`
        docker build -t $REGISTRY/$REPOSITORY1:$VERSION1 frontend
        docker push $REGISTRY/$REPOSITORY1:$VERSION1
        VERSION2=`jq -r '.version' products/package.json`
        docker build -t $REGISTRY/$REPOSITORY2:$VERSION2 products
        docker push $REGISTRY/$REPOSITORY2:$VERSION2
        VERSION3=`jq -r '.version' shopping-cart/package.json`
        docker build -t $REGISTRY/$REPOSITORY3:$VERSION3 shopping-cart
        docker push $REGISTRY/$REPOSITORY3:$VERSION3
  
  Update-Compose:
    name: 📑 Update Docker-Compose
    runs-on: ubuntu-latest
    needs: [Build]
    steps:
    - uses: actions/checkout@v3
    - name: Update Docker-Compose
      run: |
        VERSION1=`jq -r '.version' ./frontend/package.json`
        VERSION2=`jq -r '.version' ./products/package.json`
        VERSION3=`jq -r '.version' ./shopping-cart/package.json`
        NAME1='frontend'
        NAME2='products'
        NAME3='shopping-cart'
        sed -i -- "s/REGISTRY/$REGISTRY/g" docker-compose.yaml
        sed -i -- "s/REPOSITORY1/$REPOSITORY1/g" docker-compose.yaml
        sed -i -- "s/REPOSITORY2/$REPOSITORY2/g" docker-compose.yaml
        sed -i -- "s/REPOSITORY3/$REPOSITORY3/g" docker-compose.yaml
        sed -i -- "s/VERSION1/$VERSION1/g" docker-compose.yaml
        sed -i -- "s/VERSION2/$VERSION2/g" docker-compose.yaml
        sed -i -- "s/VERSION3/$VERSION3/g" docker-compose.yaml
        sed -i -- "s/NAME1/$NAME1/g" docker-compose.yaml
        sed -i -- "s/NAME2/$NAME2/g" docker-compose.yaml
        sed -i -- "s/NAME3/$NAME3/g" docker-compose.yaml
        cat docker-compose.yaml
    - uses: actions/upload-artifact@v3
      with:
        name: docker
        path: docker-compose.yaml
        
  deploy_to_dev:
    name: 🛳 Deploy to Develop "develop"
    runs-on: ubuntu-latest
    if: ${{ github.ref == 'refs/heads/develop'}}
    needs: [Update-Compose]
    environment: develop
    steps:
    - uses: actions/checkout@v3
    - uses: actions/download-artifact@v3
      with:
        name: docker 
    - name: copy file to Aws EC2
      uses: appleboy/scp-action@master
      with:
        host: ${{ secrets.HOST_D }}
        username: ${{ secrets.USERNAME }}
        port: ${{ secrets.PORT }}
        key: ${{ secrets.KEY }}
        source: "docker-compose.yaml"
        target: "/home/ec2-user"    
    - name: Deploy Docker-Compose
      uses: appleboy/ssh-action@v0.1.7
      with:
        host: ${{ secrets.HOST_D }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.KEY }}
        port: ${{ secrets.PORT }}
        script: |
          whoami
          docker-compose up -d   
  
  deploy_to_tst:
    name: 🚀 Deploy to testing "testing"
    runs-on: ubuntu-latest
    if: ${{ github.ref == 'refs/heads/testing'}}
    needs: [Update-Compose]
    environment: testing
    steps:
    - uses: actions/checkout@v3
    - uses: actions/download-artifact@v3
      with:
        name: docker 
    - name: copy file to Aws EC2
      uses: appleboy/scp-action@master
      with:
        host: ${{ secrets.HOST_T }}
        username: ${{ secrets.USERNAME }}
        port: ${{ secrets.PORT }}
        key: ${{ secrets.KEY }}
        source: "docker-compose.yaml"
        target: "/home/ec2-user"    
    - name: Deploy Docker-Compose
      uses: appleboy/ssh-action@v0.1.7
      with:
        host: ${{ secrets.HOST_T }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.KEY }}
        port: ${{ secrets.PORT }}
        script: |
          whoami
          docker-compose up -d 