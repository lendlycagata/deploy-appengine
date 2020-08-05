How to authenticate users in a Node.js Express app
This guide provides detailed instructions on how to add user authentication via OneLogin to a Node.js application.

The application is based on Express.js and uses Passport.js to complete an OpenId Connect Authorization Code flow via OneLogin.

Follow the steps below to add user authentication.

Configure OneLogin
Configure the Node.js application to connect to OneLogin
Start the application and login, logout.
View the sample code for this guide on Github

.

1. Configure OneLogin
Create a new OpenId Connect (OIDC) application from the OneLogin Administration panel.

Add a new App

Apps

Search for OIDC and select the OpenId Connect app

Add OIDC App

Create a display name for your app and Save.

Set App Name

Set a callback url. For this example, we will use http://localhost:3000/oauth/callback to test.

Set Redirect URI

On the SSO tab, note the Client ID and Client Secret. Then change the Token Endpoint Authentication Method to POST. Click Save.

Get Client ID, Client Secret

Go to the Users section to locate your test user and assign the recently created application to that user.

Assign App

2. Configure the application to connect to OneLogin
The sample app uses the Express web framework for Node.js. Express is a minimal framework based on the model, view, controller (MVC) pattern.

To enable authentication in the app we use Passport which is a popular Express middleware. It supports many different modes of authentication through what they call a Strategy. In this case we will be making use of an OpenId Connect Strategy which will handle both the initial request to OneLogin and the backend request to switch the authorization code for an access token.

The majority of the code and Passport Strategy for this sample is located in the app.js.

Setup environment variables
We will store our OIDC application credentials as environment variables.

First rename the .env.sample file to .env.

Then, set the OIDC_CLIENT_ID and OIDC_CLIENT_SECRET to the values obtained in the previous step.


OIDC_BASE_URI=https://openid-connect.onelogin.com/oidc
OIDC_CLIENT_ID=ddf97d-xxx-xxxx-e92920
OIDC_CLIENT_SECRET=f021f63bxxxx71f2c6a90c9b
OIDC_REDIRECT_URI=http://localhost:3000/oauth/callback
When the application starts it will use dotenv to load the environment variables and then use them to configure the OpenId Connect Strategy.


// Configure the OpenId Connect Strategy
// with credentials obtained from OneLogin
passport.use(new OneLoginStrategy({
  issuer: process.env.OIDC_BASE_URI,
  clientID: process.env.OIDC_CLIENT_ID,
  clientSecret: process.env.OIDC_CLIENT_SECRET,
  authorizationURL: `${OIDC_BASE_URI}/auth`,
  userInfoURL: `${OIDC_BASE_URI}/me`,
  tokenURL: `${OIDC_BASE_URI}/token`,
  callbackURL: process.env.OIDC_REDIRECT_URI,
  passReqToCallback: true
},
function(req, issuer, userId, profile, accessToken, refreshToken, params, cb) {

  console.log('issuer:', issuer);
  console.log('userId:', userId);
  console.log('accessToken:', accessToken);
  console.log('refreshToken:', refreshToken);
  console.log('params:', params);

  req.session.accessToken = accessToken;

  return cb(null, profile);
}));
Install dependencies
Use NPM to install all of the dependencies for this sample project. The main things to note here are that were using Passport.js and an OpenId Connect strategy.


npm install
3. Start the application
It’s time to start the application and test our authentication flow.


npm start
This command makes the application available to test on http://localhost:3000/.

Login
Click Login to start the authentication flow.

Start App

This triggers a GET request against the /login route of your application and redirects you to secure login page hosted by OneLogin.


// Initiates an authentication request with OneLogin
// The user will be redirect to OneLogin and once authenticated
// they will be returned to the callback handler below
app.get('/login', passport.authenticate('openidconnect', {
  successReturnToOrRedirect: "/",
  scope: 'profile'
}));
Enter username, password, and possibly MFA, depending on your security policy configuration in OneLogin.

Login

Once authentication is complete, you’re redirected back to the /callback route of your local application and provided with an authorization code.

The /callback route passes the authorization code into Passport, which sends a POST request to OneLogin and exchanges the code for an Access Token.


// Callback handler that OneLogin will redirect back to
// after successfully authenticating the user
app.get('/oauth/callback', passport.authenticate('openidconnect', {
  callback: true,
  successReturnToOrRedirect: '/users',
  failureRedirect: '/'
}));
Login Success!

Login Success

Logout
Clicking the logout link makes a request to the /logout endpoint. The logout endpoint will terminate the users local session and make a request to OneLogin to revoke the access token.


// Destroy both the local session and
// revoke the access_token at OneLogin
app.get('/logout', function(req, res){
  request.post(`https://openid-connect.onelogin.com/oidc/token/revocation`, {
    'form':{
      'client_id': process.env.OIDC_CLIENT_ID,
      'client_secret': process.env.OIDC_CLIENT_SECRET,
      'token': req.session.accessToken,
      'token_type_hint': 'access_token'
    }
  },function(err, response, body){
    console.log('Session Revoked at OneLogin');
    res.redirect('/');
  });
});
