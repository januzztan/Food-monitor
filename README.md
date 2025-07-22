# Web application for monitoring package for food preservation

## Setup steps

1. Install [node.js](https://nodejs.org/en/download) using installer

2. Run in project cd

Install firebase

```sh
npm install firebase
```

Install tailwind css
```sh
npm install tailwindcss @tailwindcss/vite
```

Install highcharts library
```sh
npm install highcharts --save
```

3. Add .env.local file to root folder

4. Add to .gitignore file 
```sh
# .gitignore
.env.local
.env.*.local
```

## Hosting Setup

Install firebase tools 
```sh
npm install -g firebase-tools
```

Login to Firebase account (make sure to be added to the firebase project)
```sh
firebase login
```
*will be redirected to login to firebase account*

Navigate to project root folder
```sh
firebse init hosting
```
Follow the steps:  
Are you ready to proceed? Yes  
What do you want to use as your public directory? (public) dist  
Configure as a single-page app (rewrite all urls to /index.html)? Yes  
Set up automatic builds and deploys with GitHub? No  
Overwrite existing files? No  

Build Vue project
```sh
npm run build
```

Host website
```sh
firebase deploy --only hosting
```
webpage should be hosted with link presented in cmd *(Remember to refresh the webpage after clicking on link)*

## Recommended IDE Setup

[VSCode](https://code.visualstudio.com/)
