```bash
yarn 
yarn start
```

It will install all the dependencies and run the app in localhost:3000

<hr>

# Google Drive upload w/ Google API in React

## Objective

To save text data as a Google Document in your drive. 

## Main Steps

1. Authorise your app and enable google api
2. Writing the code for user authentication 
3. Finally, for saving the uploading the data in google drive

### Authorise your app and enable google api

The google docs on this step is enough to get through this step. 
Here is the [link](https://developers.google.com/drive/api/v3/enable-drive-api)

Some key points to remember:
1. First create the API key in the credentials and then the Oauth Client Id
2. Make sure you specify the website from which we will sending the requests, else you will receive CORs issue. Also you cannot use IP, if you working on localhost specify ```localhost:<PORT>```


After this step you will need two things:
1. API_KEY
2. Client_ID

### Code for user Authentication

We will be using OAuth2.0 cause that's the only thing google allows. 
Let's explain what will happen in this step.

1. User will Sign In
2. User will be asked if he/she authorises this app 
3. Once user gives a consent we will recieve a token and make further request. 

Regarding oauth2.0 there is a lot of theory [here](https://developers.google.com/identity/protocols/oauth2)

Coming to implementation:

We need to specify some script so that we can use the ```gapi.client```. Now if you are in your local machine using ```gapi.client``` may give you ```undefined ```.  Instead you ```window.gapi.client```. 
There is one github issue regarding [this](https://github.com/google/google-api-javascript-client/issues/46).

Coming back in react we will append this script to the body inside the ```componentDidMount()``` function

```javascript
componentDidMount(){
    var script = document.createElement('script');
    script.onload=this.handleClientLoad;
    script.src="https://apis.google.com/js/api.js";
    document.body.appendChild(script);
  }
```
Function handleClientLoad will load the ```gapi.client``` for us.

```javascript
handleClientLoad = ()=>{
    window.gapi.load('client:auth2', this.initClient);
  }
```
As a callback we specify ```initClient``` where we initialise the ```gapi.client```
The call to ```gapi.client.init``` specifies the following fields:

1. **API_KEY** and **CLIENT_ID** : These specify your application's authorisation credentials. We have got these from the previous step. 
2. **Scope**: It specifies a space-delimited list of access scopes that correspond to the resources that your application could access on the user's behalf. Here is a [list of scopes](https://developers.google.com/identity/protocols/oauth2/scopes#drive). For our purpose we would need this  ```https://www.googleapis.com/discovery/v1/apis/drive/v3/rest```. 
3.  **DiscoveryDocs**: It identifies a list of API Discovery documents that your application uses. In this example, the code retrieves the discovery document for version 3 of the Google Drive API, 
```https://www.googleapis.com/discovery/v1/apis/drive/v3/rest```

Put these at the top
```javascript
var  SCOPE  =  'https://www.googleapis.com/auth/drive.file';
var  discoveryUrl  =  'https://www.googleapis.com/discovery/v1/apis/drive/v3/rest';
```
So, now let's define out ```initClient``` function. 

```javascript
initClient = () => {
    try{
      window.gapi.client.init({
          'apiKey': "<YOUR API KEY>",
          'clientId': "<YOUR CLIENT ID>",
          'scope': SCOPE,
          'discoveryDocs': [discoveryUrl]
        }).then(() => {
          this.setState({
            googleAuth: window.gapi.auth2.getAuthInstance()
          })
          this.state.googleAuth.isSignedIn.listen(this.updateSigninStatus);  


         document.getElementById('signin-btn').addEventListener('click', this.signInFunction);
         document.getElementById('signout-btn').addEventListener('click', this.signOutFunction);
          
      });
    }catch(e){
      console.log(e);
    }
  }

```

Once the client is initialised we get an Auth Instance and save it in a state variable, ```googltAuth ```. The ```updateSigninStatus ``` function is a listener that listens for changes to the user's authorization status. Also we add some functions to the sign in and sign out buttons. So, before moving on we need to specify our states. 

```javascript
  state = {
    name: '',
    googleAuth: ''
  }
```

The ```name ``` variable is for other purpose, we will come later. Now if the user clicks on sign in button the ```signInFunction ``` will be triggered. 

```javascript
signInFunction =()=>{
    this.state.googleAuth.signIn();
    this.updateSigninStatus()
  }
```

Since after sign in the state changes we will explicitly call the ```updateSigninStatus()``` function.  The signout function does something very similar. 


```javascript
signOutFunction =()=>{
    this.state.googleAuth.signOut();
    this.updateSigninStatus()
  }
```

Now let's come to ```updateSignStatus()```. All it does is fetch some user details(here, the name and this is where we use the name state variable). 
```javascript
updateSignStatus = async ()=>{
    var user = this.state.googleAuth.currentUser.get();
    if (user.wc == null){
      this.setState({
        name: ''
      });
    }
    else{
      var isAuthorized = user.hasGrantedScopes(SCOPE);
      if(isAuthorized){
        this.setState({
          name: user.Ot.Cd
        });
        //we will put the code of the third step here
      }
    }
  }
```

Important thing to note here is that ```isAuthorized ``` is true only if the user grants the permissions to the app. Now once we are done upto here, now we can move to the final step of uploading the file. 

### Uploading the data in google drive

For uploading the data we have various methods. In our case we will use the Multipart method because we will not just create a file but also specify the meta data as well. All the code snippets in this step will be inside the region specified in the above step.

The steps mentioned in the google docs are:
1. Create a POST request to the method's /upload URI with the query parameter of uploadType=multipart:

    ```POST https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart```

2. Create the body of the request. Format the body according to the multipart/related content type [RFC 2387], which contains two parts:

    - Metadata. The metadata must come first and must have a Content-Type header set to application/json; charset=UTF-8. Add the file's metadata in JSON format.
    - Media. The media must come second and must have a Content-Type header of any MIME type. Add the file's data to the media part.

    Identify each part with a boundary string, preceded by two hyphens. In addition, add two hyphens after the final boundary string.

3. Add these top-level HTTP headers:

    - Content-Type. Set to multipart/related and include the boundary string you're using to identify the different parts of the request. For example: Content-Type: multipart/related; boundary=foo_bar_baz
    - Content-Length. Set to the total number of bytes in the request body.


Send the request.

So, let's create the metadata of the file

```javascript
var fileName='mychat123';
var fileData='this is a sample data';
var contentType='text/plain'
var metadata = {
      'name': fileName,
      'mimeType': contentType
};

```

You can change the ```fileName``` and ```fileData ``` and also change the ```contentType ``` accordingly, it will hold the MIME type of the data you will upload to drive.

Now the multipart body. It follows a particular standardisation, you can read more about it [here](https://tools.ietf.org/html/rfc2387)

Without going into much details just copy the following. 
```javascript
const boundary='<ANY RANDOM STRING>'
const delimiter = "\r\n--" + boundary + "\r\n";
const close_delim = "\r\n--" + boundary + "--";
```

here ```boundary ``` will differentiate between the various part of the request body. 

```javascript
var multipartRequestBody =
          delimiter +
          'Content-Type: application/json; charset=UTF-8\r\n\r\n' +
          JSON.stringify(metadata) +
          delimiter +
          'Content-Type: ' + contentType + '\r\n\r\n' +
          fileData+'\r\n'+
          close_delim;
```

This is a format and it needs to be followed. After this all we are left with is sending the request which we will do using the ```gapi.client.request``` this will handle the Auth Token automatically. 

```javascript
var request = window.gapi.client.request({
            'path': 'https://www.googleapis.com/upload/drive/v3/files',
            'method': 'POST',
            'params': {'uploadType': 'multipart'},
            'headers': {
              'Content-Type': 'multipart/related; boundary=' + boundary + ''
            },
            'body': multipartRequestBody});
request.execute(callback);
```

Now we are DONE!!. 
To compile it all this was my App.js 

```javascript
import React, { Component } from 'react';

var SCOPE = 'https://www.googleapis.com/auth/drive.file';
var discoveryUrl = 'https://www.googleapis.com/discovery/v1/apis/drive/v3/rest';


class App extends Component {
  state = {
    name: '',
    googleAuth: ''
  }
  componentDidMount(){
    var script = document.createElement('script');
    script.onload=this.handleClientLoad;
    script.src="https://apis.google.com/js/api.js";
    document.body.appendChild(script);
  }
  

  initClient = () => {
    try{
      window.gapi.client.init({
          'apiKey': "",
          'clientId': "",
          'scope': SCOPE,
          'discoveryDocs': [discoveryUrl]
        }).then(() => {
          this.setState({
            googleAuth: window.gapi.auth2.getAuthInstance()
          })
          this.state.googleAuth.isSignedIn.listen(this.updateSigninStatus);  
         document.getElementById('signin-btn').addEventListener('click', this.signInFunction);
         document.getElementById('signout-btn').addEventListener('click', this.signOutFunction);
          
      });
    }catch(e){
      console.log(e);
    }
  }


  signInFunction =()=>{
    this.state.googleAuth.signIn();
    this.updateSigninStatus()
  }

  signOutFunction =()=>{
    this.state.googleAuth.signOut();
    this.updateSigninStatus()
  }

  updateSigninStatus = ()=> {
    this.setSigninStatus();
  }

  
  setSigninStatus= async ()=>{
    var user = this.state.googleAuth.currentUser.get();
    console.log(user)
    if (user.wc == null){
      this.setState({
        name: ''
      });
    }
    else{
      var isAuthorized = user.hasGrantedScopes(SCOPE);
      if(isAuthorized){
        this.setState({
          name: user.Ot.Cd
        });
        const boundary='foo_bar_baz'
        const delimiter = "\r\n--" + boundary + "\r\n";
        const close_delim = "\r\n--" + boundary + "--";
        var fileName='mychat123';
        var fileData='this is a sample data';
        var contentType='text/plain'
        var metadata = {
          'name': fileName,
          'mimeType': contentType
        };

        var multipartRequestBody =
          delimiter +
          'Content-Type: application/json; charset=UTF-8\r\n\r\n' +
          JSON.stringify(metadata) +
          delimiter +
          'Content-Type: ' + contentType + '\r\n\r\n' +
          fileData+'\r\n'+
          close_delim;

          console.log(multipartRequestBody);
          var request = window.gapi.client.request({
            'path': 'https://www.googleapis.com/upload/drive/v3/files',
            'method': 'POST',
            'params': {'uploadType': 'multipart'},
            'headers': {
              'Content-Type': 'multipart/related; boundary=' + boundary + ''
            },
            'body': multipartRequestBody});
        request.execute(function(file) {
          console.log(file)
        });
      }
    }
  }

  handleClientLoad = ()=>{
    window.gapi.load('client:auth2', this.initClient);
  }
  render() {
    return (
      <div className="App">
        <div>UserName: <strong>{ this.state.name}</strong></div>
        <button id="signin-btn">Sign In</button>
        <button id="signout-btn">Sign Out</button>
      </div>
    );
  }
}

export default App;

```

I had to go through a lot of documentation of Google APIs to be able to come to this. I tried many other approaches but at this was the one that worked for me. If you are stuck at any point do check out the [Oauth Playground](https://developers.google.com/oauthplayground/)

<hr>

## Refs
1. [Upload Files Doc](https://developers.google.com/drive/api/v3/manage-uploads)
2. [File Create](https://developers.google.com/drive/api/v3/reference/files/create)
3. [JSON API multipart](https://cloud.google.com/storage/docs/json_api/v1/how-tos/multipart-upload)
4. [MIME Multipart/Related Content-type](https://tools.ietf.org/html/rfc2387)
5. [Other MIME types](https://developers.google.com/drive/api/v3/mime-types)