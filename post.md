## Introduction

In this guide you would learn how to use open technology stacks and a set of cloud services & platforms (that provide free tier) to create a multi-tenant, real time, secure and scalable solution.

#### Multitenancy
[Multitenancy](http://en.wikipedia.org/wiki/Multitenancy) in simple words mean that your software architecture should be able to support multiple customers or tenants. A multi-tenant application would not require you to run separate or customized instances for different customer organizations. For the purpose of this guide the multitenancy is desired from the logical separation of data and does not require physical separation of the data.   

#### Real Time
For the puposes of this guide, real time means that client applications would be able to receive new or updated information without explicitly polling the servers. JavaScript front end applications make use of websockets to achieve injection of new content without the application layer performing polling.

#### Scalable
The solution should be able to handle the load as the number of users of the solution grow. We would make use of auto scaling capabilities of various cloud platforms with mimumum configuration or operational head aches from our side, however we must strive to design our data structures to be more performance aware.

#### Secure
Since it would be a multi-tenant solution, it becomes even more important to understand the security implications of our design choices. We would create a solution that would rely on the role based security.

### YATODO (Yet Another TODO :)  )

The solution that we would create would be a TODO cloud service and is primarily intended to be used from a mobile application. Here are some high level requirements :

* Should allow free registration of an user account

* Should allow the user to register an organization (a tenant). The user who registers an organization would become its admin

* Should allow an admin to create staff records for his organization

* Should allow admin to send invites to other registered users and upon verification that user would be registered as the staff with the specified role (admin or member)

* Should allow a user to be a staff of multiple organizations

* Should allow staff of an organization to manage (create, read, update & delete) his/her todo items.

* Should allow admin of an organization to create todo-items for his staff

* Should allow admin to read the todo items of all his staff

* Should not allow user with member role to read/write todo items of other  members of his and/or other organizations

* Should not allow admin of an organization to read/write todo items of other organizations where he is not registered and/or is a member 


## Available Technology Stacks

Chosing which techonology is appropriate for a solution is always a tough nut to crack. Every technology/platform has its pros and cons and I can guarantee that we will miss the features of the ones that we would not chose for our solution :) 

As per our objectives we would want to stay away from managing the server operational aspects of our solution as much as possible. This can be achieved using a relatively newer branch of cloud computing called Backend as a Service (BaaS). There are many vendors who offer solutions in this space, the ones that we are going to discuss in brief are [Parse](http://www.parse.com) and [Firebase](http://www.firebase.com).

### Parse

Parse (now part of Facebook) is one of the earlier players in BaaS space. Their offerings include
    - Cloud database
    - RESTful apis
    - SDKs for JavaScript, .NET, PHP, Java, C#
    - Push Notifications
    - Cloud functions
    - Web hooks
    - User Authentication
    - Authorization using ACL
    
As you can see they provide an impressive set of services. However, there is no real time support from their platform itself. If you want one, then you can achieve that by making use of real time providers ([Pusher](https://pusher.com/), [pubnub](https://www.pubnub.com/) etc) and doing integration using the Parse's cloud modules/functions.

### Firebase

Firebase (now part of Google) also offers cloud database with support for user registration and authentication. Instead of having an ACL on the data objects, you get to define the access control using a security language. The main benefit of their approach is that it helps in managing the security rules much more eligantly and they all are in one place. Having them decopuled from data is a great design choice. The most notable feature of firebase is the real time aspect of their offering. They have made it almost trivial to develop real time applications.

## The fun begins

Even though firebase lacks push notifications and the capability to run cloud functions, their real time offering is very attractive and hence for this guide we are going to use them as our cloud database provider.

The code base developed in this guide is available here - [https://github.com/ksachdeva/YATODO](https://github.com/ksachdeva/YATODO)

I have divided the development of the solution in stages and each stage corresponds to a branch in git. A very quick re-cap for git and branches for you :

```bash
# to see all the remote branches
$ git branch -r
  origin/HEAD -> origin/master
  origin/master
  origin/ninja-stage-1
  origin/ninja-stage-2
  origin/ninja-stage-3
  origin/ninja-stage-4
  origin/ninja-stage-5
  origin/ninja-stage-6
```

```bash
# to create a local branch from remote branch you would do following
$ git checkout -b ninja-stage-1 origin/ninja-stage-1
```

### Stage 1

[https://github.com/ksachdeva/YATODO/tree/ninja-stage-1](https://github.com/ksachdeva/YATODO/tree/ninja-stage-1)

As we discussed earlier, firebase has a notion of security rules using which you can specify permissions for data read and write. Writing those security rules could be bit challenging and hence we are going to spend a lot of time discussing the rules. As the best way to explain is by showing the code so here we go -

```javascript
// This is the default state of data at firebase 
{}

// and this is the default state of rules
{
  "rules" : {
    ".read" : true,
    ".write" : true
  }
}
```

As part of its offerings, firebase has a built-in user authentication support. You could use their username-password based authentication or perform (custom) authentication using web tokens. We would use their username-password authentication system and would not talk about custom authentication in this guide. Also the username is actually the email of the user. 

The default rules mean that anyone can read or write our data, 
so the first thing that we want to do is turn off the default permissions

```javascript
{
    "rules" : {
        ".read" : false,
        ".write" : false
    }   
}
```

Now let's work on the data structure to store users' information, this information could be his profile (name, email, address, phone etc) and/or other data structures that may be required. 

When you use the username-password authentication system, the user gets a unique identification in your firebase database. The uid for username-password authentication has 'simplelogin:X' format where X is a number.

Here is our desired structure for user's data whose *uid* is *simplelogin:1*

```javascript
{
    "users" : {
        "simplelogin:1" : {
            "profile" : {
                "name" : "John Doe",
                "email" : "john@doe.com"
            }
        }
    }
}
```

As per the current state of our rules we simply do not allow any read or write. It is time for us to start granting some permissions. 

```
Requirement :
-----------
A registerd and authenticated user should be able to read and write his/her profile.
```
```javascript
// contents of test1_rules.json
{
  "rules": {
    ".read": false,
    ".write": false,

    "users": {
      "$userId": {
        "profile": {
          ".read": "(auth != null) && ($userId === auth.uid)",
          ".write": "(auth != null) && ($userId === auth.uid)"
        }
      }
    }
  }

}

```

So here is what this updated rules definition is doing - 

* it would allow reading & writing of the profile object at path users/$userId/ if the authenticated user has an auth.uid that is equal to $userId

* auth.uid is a firebase magic variable that contains the uid of the authenticated user

* $xxx (or here $userId) is a variable that will have dynamic value. We would put the uid of the user as the value at the time of profile creation

Now that you have the rules defined, you can log in to your firebase account, create an instance if you do not have any, go to the dashboard and paste the above security rules. Below is an image showing the location where you would post these rules :

![Rules](http://i.imgur.com/qJ84d5N.png)

The next question is how to validate if our rules would work for our desired data structure. Firebase provides a simulator at their dashboard that you can use to try these rules. 

When you will click on the Simulator tab you would see two portions. The upper portion would let you select the authentication type & sample credential and the lower portion is about urls and data that you would use to simulate read and write.

Now we would like to use email-password based authentication but this choice does not appear and is bit confusing in the simulator. The image below is showing how you would specify the email-password based authentication

![Simulate Authentication](http://i.imgur.com/z2ad5Hb.png)

Clicking on Authenticate button will show "Authenticated" message. *Please note that to use simulator you do not need to create/register users*

The next step is to simulate a read operation

![Simulate Read](http://i.imgur.com/rGwnv4q.png)

While we are here let's try the case that should not work. We are logged as simplelogin:1 but we would try to read the profile for simplelogin:2.

![Simulate Read Denied](http://i.imgur.com/Nrx1oNr.png)

And finally let see how to test the write operations

![Simulate Write](http://i.imgur.com/JdggBg5.png)

The simulator in the dashboard is great but is not necessarily a friendly tool when our application will grow bigger after addition of more data structures and their corresponding rules will be defined. Therefore, we would like to have the verification of rules written in forms of test cases that can be exercised manually or using continuous integration.

Here is the directory structure that I have come up for our solution, the name of directories should be clear enough to understand their purpose.

```
.
└── YATODO
    ├── LICENSE
    ├── README.md
    └── src
        ├── client
        │   └── mobile
        └── server
            └── firebase
                └── test
```

The test cases that we want to write should be such that they should be indpendent of each other which implies that before every test case we would want to upload the rules and desired data (if any) for that test case. 

Also we would want to pre-create some user accounts that we would use of testing. Go back to the dashboard again, make sure that the email-password based authentication is enabled as shown below
    ![Enable Auth](http://i.imgur.com/WaIri2F.png)
    
Add some user accounts (would be a good idea to use real email addresses even though the dashboard will allow you to specify fake email addresses e.g abc@abc.com)
    ![Add user](http://i.imgur.com/tnLKCdu.png)
    
The test framework that we would use is called [mocha](http://mochajs.org) and an assertion library called [chai](http://chaijs.com/). 

We would write our test projects such that all the configuration related things can be put in one file so that you can fetch [YATODO Repo](https://github.com/ksachdeva/YATODO.git) and only provide .env file with your secret keys and details about users. 

```bash
# here is what you need to do to fetch my repo
npm install -g mocha
git clone https://github.com/ksachdeva/YATODO.git
git checkout -b ninja-stage-1 origin/ninja-stage-1
cd YATODO/src/server/firebase/test
npm install
touch .env
```

```bash
# do this if you want to create everything by yourself from scratch
npm install -g mocha   
cd YATODO/src/server/firebase/test
npm init    
npm install firebase --save
npm install chai --save
npm install node-env-file --save
npm install request-promise --save
touch .env
touch fbutils.js
touch test1.js
touch data/test1_data.json
touch data/test1_rules.json
```

```javascript
// contents of .env file
FB_NAME=???
FB_SECRET_KEY=???

TEST1_EMAIL=???
TEST1_PWD=???

TEST2_EMAIL=???
TEST2_PWD=???

TEST3_EMAIL=???
TEST3_PWD=???
```

Please replace ??? with the values that correspond to your firebase instance.

Since we are supposed to systematically load new rules and data as each test case(s) would require, I have put that utility method in fbutils.js

```javascript,linenums=true
// contents of fbutils.js
var rp = require('request-promise');
var env = require('node-env-file');

env(__dirname + '/.env');

var FB_NAME = process.env.FB_NAME;
var FB_SECRET_KEY = process.env.FB_SECRET_KEY;

var baseUrl = 'https://' + FB_NAME + "/";

var rulesUrl = baseUrl + '.settings/rules.json?auth=' + FB_SECRET_KEY;
var rootDataUrl = baseUrl + '.json?auth=' + FB_SECRET_KEY;

exports.BASE_URL = baseUrl;

exports.TEST1_ACCOUNT = {
  "email": process.env.TEST1_EMAIL,
  "pwd": process.env.TEST1_PWD
};

exports.TEST2_ACCOUNT = {
  "email": process.env.TEST2_EMAIL,
  "pwd": process.env.TEST2_PWD
};

exports.TEST2_ACCOUNT = {
  "email": process.env.TEST2_EMAIL,
  "pwd": process.env.TEST2_PWD
};

exports.loadRulesAndData = function(rules, newData) {

  return rp({
      url: rulesUrl,
      method: 'PUT',
      json: rules
    })
    .then(function() {
      return rp({
        url: rootDataUrl,
        method: 'PUT',
        json: newData
      });
    });

};
```

Now let's look at our test1.js file (note that data\test1_data.json & data\test1_rules.json contain the data that we used for simulation earlier so I would not paste them here)

```javascript,linenums=true
var fbUtils = require('./fbutils.js');

var Firebase = require('firebase');
var chai = require('chai');

var expect = chai.expect;
var should = chai.should();

var sampleRules = require('./data/test1_rules.json');
var sampleData = require('./data/test1_data.json');

var FB_BASE_URL = fbUtils.BASE_URL;

var TEST1_ACCOUNT = fbUtils.TEST1_ACCOUNT;
var TEST2_ACCOUNT = fbUtils.TEST2_ACCOUNT;
var TEST3_ACCOUNT = fbUtils.TEST3_ACCOUNT;

describe('Stage1 Test Cases', function() {

  beforeEach(function(done) {
    // this code will run before each test case
    // so that we have the clean test data
    // on firebase server

    // upload rules & data
    fbUtils.loadRulesAndData(sampleRules, sampleData)
      .catch(console.error)
      .finally(done);
  });

  it('should allow creating users/simplelogin:1 for user simplelogin:1', function(done) {

    var ref = new Firebase(FB_BASE_URL);

    // authenticate the user
    ref.authWithPassword({
      email: TEST1_ACCOUNT.email,
      password: TEST1_ACCOUNT.pwd
    }, function(error, authData) {

      // once auth is done now create
      // the profile object
      var profileRef = ref.child("users/" + authData.uid + "/profile");

      var objectToWrite = {
        email: TEST1_ACCOUNT.email,
        name: 'First Tester'
      };

      profileRef.set(objectToWrite, function(error) {
        should.not.exist(error);
        done();
      });

    });

  });

  it('should allow reading profile of users/simplelogin:1 for user simplelogin:1', function(done) {

    var ref = new Firebase(FB_BASE_URL);

    ref.authWithPassword({
      email: TEST1_ACCOUNT.email,
      password: TEST1_ACCOUNT.pwd
    }, function(error, authData) {

      var profileRef = ref.child("users/" + authData.uid + "/profile");

      // try to read the value only once
      profileRef.once('value', function(snapshot) {
        should.exist(snapshot);
        done();
      }, function(errorObj) {
        should.not.exist(errorObj);
        done();
      });

    });

  });

  it('should not allow creating users/simplelogin:2 by user simplelogin:1', function(done) {

    var ref = new Firebase(FB_BASE_URL);

    ref.authWithPassword({
      email: TEST1_ACCOUNT.email,
      password: TEST1_ACCOUNT.pwd
    }, function(error, authData) {

      var profileRef = ref.child("users/" + 'simplelogin:2' + "/profile");

      var objectToWrite = {
        email: TEST1_ACCOUNT.email,
        name: 'First Tester'
      };

      profileRef.set(objectToWrite, function(error) {
        should.exist(error);
        done();
      });

    });

  });

  it('should not allow reading users/simplelogin:1 by user simplelogin:2', function(done) {

    var ref = new Firebase(FB_BASE_URL);

    ref.authWithPassword({
      email: TEST2_ACCOUNT.email,
      password: TEST2_ACCOUNT.pwd
    }, function(error, authData) {

      var profileRef = ref.child("users/" + 'simplelogin:1' + "/profile");

      profileRef.once('value', function(snapshot) {
        should.not.exist(snapshot);
        done();
      }, function(errorObj) {
        should.exist(errorObj);
        done();
      });

    });

  });

});

```

Time to run our test cases

```bash
# run the test cases using following command
$ mocha test1.js

  Stage1 Test Cases
    ✓ should allow creating users/simplelogin:1 for user simplelogin:1 (574ms)
    ✓ should allow reading profile of users/simplelogin:1 for user simplelogin:1 (574ms)
FIREBASE WARNING: set at /users/simplelogin:2/profile failed: permission_denied
    ✓ should not allow creating users/simplelogin:2 by user simplelogin:1 (608ms)
    ✓ should not allow reading users/simplelogin:1 by user simplelogin:2 (608ms)


  4 passing (3s)
```


### Stage 2

[https://github.com/ksachdeva/YATODO/tree/ninja-stage-2](https://github.com/ksachdeva/YATODO/tree/ninja-stage-2)

We spent lot of time in stage 1 to learn first about using the simulation and then setting up a test project for us. Now we would work out some more complicated rules and data structure for our solution.

```
Requirement:
------------
Should allow a user to create/register an organization and he should be marked as the admin of that organization
```

Here is an attempt at trying to model the data structure for organizations

```javascript,linenums=true
{
    "users" : {
        "simplelogin:1" : {
            "profile" : {
                "name" : "John Doe",
                "email" : "john@doe.com"
            }
        }
    },
    
    "organizations" : {
        "-uniqueOrgId_1" : {
            "about" : {
                "address" : "1200, 6th Street, Austin, TX",
                "name" : "A great Company",
                "phone" : "512-111-1111"
            }
        },
        
        "-uniqueOrgId_2" : {
            // .... 
        }
    }
}
```

Issues with the above data model :

- does not specify who is the admin of this organization
- normally we would want the user to easily obtain the list of organizations to which he belongs and his roles in those organizations.

Refining our data model as :

```javascript,linenums=true
//
// role vaule of 1 => A member
// role value of 5 => An admin
{
    "users" : {
        "simplelogin:1" : {
            "profile" : {
                "name" : "John Doe",
                "email" : "john@doe.com"
            },
            "organizations" : {
                "-uniqueOrgId_1" : {
                    "name" : "A great Company",
                    "role" : 5
                }
            }
        }
    },
    
    "organizations" : {
        "-uniqueOrgId_1" : {
            "about" : {
                "address" : "1200, 6th Street, Austin, TX",
                "name" : "A great Company",
                "phone" : "512-111-1111"
            },
            
            "staff" : {
                "-uniqueStaffId_1" : {
                    "name" : "John Doe",
                    "email" : "john@doe.com",
                    "role"  : 5
                }
            }
        },
        
        "-uniqueOrgId_2" : {
            // .... 
        }
    }
}
```

So here we have created "organizations" node that groups the organizations. An organization is identified using a unique key for e.g for "A great company", it is "-uniqueOrgId_1". Here I have picked a readable name, in reality its a random string of characters that firebase generates and they are guaranteed to be unique for your firebase account. We also updated the users/$userId node to have the list of organizations to which user belong.

If you are coming from traditional SQL (RDBMS) background then above would look very strange. You would say (and rightly so) that there is lot of data duplication. While it is strange, in the NoSQL world it is considered okay to *de-normalize* the data to speed up read operations. Here is a link to a great tutorial ["Structuring your Firebase Data correctly for a Complex App"](https://www.airpair.com/firebase/posts/structuring-your-firebase-data) that explains in detail the motivation behind de-normalization with examples.

That being said, the data duplication is not really the thing to worry about here, the questions we should be asking ourselves are -

- firebase allowed user to specify an email & password to register. There was no email verification => anyone could claim some one elses email address !!

- an admin should be allowed to add staff members and assign them roles but who should create the entry for him and more importantly a "user" should not be able to add entries of organizations for himself !! ... he should be limited to only updating the profile now. However, he should be allowed to read his list of organizations.

The above mentioned concerns are the security issues, unfortunately at this time firebase does not have the support for "Email Verification" when user creates/register an account. For the second issue, writing the organization entries in "users" data structure must be done on the server. In other words, it should not be driven by the client application.

Wait, what !! .. Did you say "Server" ?. You may say - I would like to make use of BaaS (Backend as a Service), write a mobile app and would ideally want to stay away from running/managing a server. These was an important objective for us.

Now, dear reader, the issues are very real and they do require some business logic to be run on the server but I do not disagree with your objective of staying away from running a server farm and managing them. 

Fortunately there are solutions that you could employ today and there is a promise of improvements from firebase itself that would address some concerns like above. I would describe the solution for these issues but alas if I do it right now it would steer us away from the exercise of defining the data model and corresponding security rules. 

For now we would assume following updates to our data model. Note the comments in the data mentioned below :

```javascript,linenums=true
//
// role vaule of 1 => A member
// role value of 5 => An admin
{
    "users" : {
        "simplelogin:1" : {
            "profile" : {
                "name" : "John Doe",
                "email" : "john@doe.com"
            },
            
            // below mentioned data is added by code
            // running on the server
            // Also below information can be read by the user (John Doe)
            // but he must not be able to write/update it
            
            "verified" : true,
            "organizations" : {
                "-uniqueOrgId_1" : {
                    "name" : "A great Company",
                    "role" : 5
                }
            }
        }
    },
    
    "organizations" : {
    
        // The org, with first entry for staff would be 
        // created by business logic running on server
        // Later the admin of this org (John Doe) should be able
        // to update the "about" and add new staff records
    
        "-uniqueOrgId_1" : {
            "about" : {
                "address" : "1200, 6th Street, Austin, TX",
                "name" : "A great Company",
                "phone" : "512-111-1111"
            },
            
            "staff" : {
                "-uniqueStaffId_1" : {
                    "name" : "John Doe",
                    "email" : "john@doe.com",
                    "role"  : 5
                }
            }
        },
        
        "-uniqueOrgId_2" : {
            // .... 
        }
    }
}
```

If you read the comments in the above data block, they imply that we need to now work on our security rules. Here is what the rules should be :

```javascript
// contents of test2_rules.json
{
  "rules": {
    ".read": false,
    ".write": false,

    "users": {
      "$userId": {
        "profile": {
          ".read": "(auth != null) && ($userId === auth.uid)",
          ".write": "(auth != null) && ($userId === auth.uid)"
        },
        
        "organizations" : {
            ".read" : "(auth != null) && ($userId === auth.uid)",
            ".write" : false  // <--- NOTE THIS
        }
      }
    },
    
    "organizations": {
      "$organization": {

        "about": {
          ".read": "root.child('users').child(auth.uid).child('organizations').hasChildren([$organization])",
          ".write": "root.child('users').child(auth.uid).child('organizations').child($organization).child('role').val() === 5"
        },

        "staff": {
          "$staffId": {
            ".read": "root.child('users').child(auth.uid).child('organizations').child($organization).child('role').val() === 5",
            ".write": "root.child('users').child(auth.uid).child('organizations').child($organization).child('role').val() === 5"
          }
        }

      }
    }
  }
}
```
 
Our simpler security rules are now getting more complex. Lets address them one by one:

* Anything under users/$userId/organizations can be read provided the logged in user's auth.uid is equal to $userId. This is same as the rule for reading the profile. Because of this permission we would be able to read the organizations to which this user belong.
 
* No user should be allowed to write at location users/$userId/organizations and hence the .write has been set to false

* The "about" data structure should be readable to the members of the organization so we need to figure out if the authenticated user is a member of the organization whose 'about' is being read !. To help us with this, firebase provided another magic object called *root*. As the name implies it the root of your entire data. 
    - root.child("users") will have access to the list of users
    - root.child("users").child(auth.uid) will point to users/$userId
    - root.child("users").child(auth.uid).child('organizations') will point to users/$userId/organizations list
    - now by doing root.child ........('organizations').hasChildren($organization) we are checking if the 'organizations' contain $organization or not. This was our objective

* The "about" data structure should be updatable by the admin. The rules are similar to what we just discussed with the differnce that we need to be able to read the value of 'role' from here users/$userId/organizations/$organization/role. The usage of root.child('..').child('...') should be clear now.

* If you have understood the "about" then understanding the rules for "staff/$staffId" should not be difficult. They are same as that of .write for 'about' data structure 

We have worked out the rules now it is time to test them using our test framework. You should be easily able to add to the code from stage-1. I have created another branch (ninja-stage-2) that you should be able to run.

```bash
# if you want to write it yourself
touch data\test2_data.json
touch data\test2_rules.json
touch test2.js
```

```bash
# if you want to use my repo
git checkout -b ninja-stage-2 origin/ninja-stage-2
```

Start by copying the contents of test1.js in test2.js but make sure to update following :

```javascript
var sampleRules = require('./data/test2_rules.json');
var sampleData = require('./data/test2_data.json');
```

Here are some of the test cases (which should reflect our requirements that we established at the start of this guide)

```javascript,linenums=true
it('should allow reading organizations list of users/simplelogin:1 by user simplelogin:1', function(done) {

    var ref = new Firebase(FB_BASE_URL);

    ref.authWithPassword({
      email: TEST1_ACCOUNT.email,
      password: TEST1_ACCOUNT.pwd
    }, function(error, authData) {

      var orgRef = ref.child("users/" + authData.uid + "/organizations");

      // try to read the value only once
      orgRef.once('value', function(snapshot) {
        should.exist(snapshot);
        done();
      }, function(errorObj) {
        should.not.exist(errorObj);
        done();
      });

    });

  });

  it('should not allow reading users/simplelogin:1 organizations by user simplelogin:2', function(done) {

    var ref = new Firebase(FB_BASE_URL);

    ref.authWithPassword({
      email: TEST2_ACCOUNT.email,
      password: TEST2_ACCOUNT.pwd
    }, function(error, authData) {

      var orgRef = ref.child("users/" + 'simplelogin:1' + "/organizations");

      orgRef.once('value', function(snapshot) {
        should.not.exist(snapshot);
        done();
      }, function(errorObj) {
        should.exist(errorObj);
        done();
      });

    });

  });

  it('should allow only admin to create new staff members', function(done) {

    var ref = new Firebase(FB_BASE_URL);

    ref.authWithPassword({
      email: TEST1_ACCOUNT.email,
      password: TEST1_ACCOUNT.pwd
    }, function(error, authData) {

      var staffRef = ref.child("organizations").child("-uniqueOrgId_1").child("staff");

      staffRef.push({
        "email": TEST2_ACCOUNT.email,
        "password": TEST2_ACCOUNT.pwd,
        role: 1
      }, function(error) {
        should.not.exist(error);
        done();
      });

    });

  });
```

### Stage 3

[https://github.com/ksachdeva/YATODO/tree/ninja-stage-3](https://github.com/ksachdeva/YATODO/tree/ninja-stage-3)


So by now we have management of staff under control and it is time to discuss the todo items. Here is a re-cap of our requirements for todo items.

```
Requirements:
------------
A member should be able to create todo items for himself but not for other members.

A member should be able to only read his todo items.

An admin should be able to write todo items for other members.

An admin should be able to read todo items of other members.
```

Based on this first we would try to define our data structure for todo items

```javascript
{
    "organizations" : {
    
        "-uniqueOrgId_1" : {
            "todos" : {
                "-uniqueStaffId_1" : {
                    "-uniquetodo_id_1" : {
                       title : "my todo 1"
                    },
                    "-uniquetodo_id_2" : {
                       title : "my todo 2"
                    }
                },
                
                "-uniqueStaffId_2" : {
                    "-uniquetodo_id_3" : {
                       title : "my todo 1"
                    },
                    "-uniquetodo_id_4" : {
                       title : "my todo 2"
                    }
                }
            }       
        }
    }
}
```

In the above data structure, we have organized the todo items using the unique ids of the staff. Because of this structure, one could fetch all the todos of all the staff members by simply reading the reference "/organizations/-uniqueOrgId_1/todos" and if intersted in reading todos of one particular staff then would read the reference url "/organizations/-uniqueOrgId_1/todos/-uniqueStaffId_1" 

Time for the rules now -

```javascript
// iteration 1
"todos": {
          ".read": false,
          "$staffId": {
            "$todoId": {
              ".read": false,
              ".write": false
            }
          }
        }
```

```javascript
// iteration 2
"todos": {
".read": "root.child('users').child(auth.uid).child('organizations').child($organization).child('role').val() === 5",
                    "$staffId": {
            "$todoId": {
              ".read": false,
              ".write": false
            }
          }
        }
```

So thanks to iteration 2 an admin should be able to read anything underneath "todos" node.

For individual todo items ($todoId) we want to make sure that a member should not be able to read or write other member's items. Now we do have information (email, role, name etc) about the logged in user but we *do not* have information about logged in staff !!!

This means that we should have "staffId" information with the user as well so lets first modify our data structure users/$userId/organization/$orgId

```javascript
// updated data structure for user
"users": {
    "simplelogin:1": {
      "profile": {
        "name": "John Doe",
        "email": "john@doe.com"
      },
      "verified": true,
      "organizations": {
        "-uniqueOrgId_1": {
          "name": "A great Company",
          "staffId": "uniqueStaffId_1", // <-- NOTE THIS
          "role": 5
        }
      }
    }
}
```
Also do not forget that this entry should be created by some business logic running on backend when the user joins the organization. 

```javascript
// iteration 3 for our rules
"todos": {
          ".read": "root.child('users').child(auth.uid).child('organizations').child($organization).child('role').val() === 5",
          "$staffId": {
            "$todoId": {
              ".read": "(root.child('users/' + auth.uid + '/organizations/' + $organization).child('role').val() === 5) || (root.child('users/' + auth.uid + '/organizations/' + $organization).child('staffId').val() === $staffId)",
              ".write": "(root.child('users/' + auth.uid + '/organizations/' + $organization).child('role').val() === 5) || (root.child('users/' + auth.uid + '/organizations/' + $organization).child('staffId').val() === $staffId)"
            }
          }
        }
```

Now that we have our rules finalized, in order to write our test cases we would want to update our sample data as well to include the staffid and create a verified account for simplelogin:2

```javascript
// updated data/test3_data.json
{
  "users": {
    "simplelogin:1": {
      "profile": {
        "name": "John Doe",
        "email": "john@doe.com"
      },
      "verified": true,
      "organizations": {
        "-uniqueOrgId_1": {
          "name": "A great Company",
          "staffId": "uniqueStaffId_1",
          "role": 5
        }
      }
    },

    "simplelogin:2": {
      "profile": {
        "name": "Jane Doe",
        "email": "jane@doe.com"
      },
      "verified": true,
      "organizations": {
        "-uniqueOrgId_1": {
          "name": "A great Company",
          "staffId": "uniqueStaffId_2",
          "role": 1
        }
      }
    }

  },

  "organizations": {
    "-uniqueOrgId_1": {
      "about": {
        "address": "1200, 6th Street, Austin, TX",
        "name": "A great Company",
        "phone": "512-111-1111"
      },

      "staff": {

        "-uniqueStaffId_1": {
          "name": "John Doe",
          "email": "john@doe.com",
          "role": 5
        },

        "-uniqueStaffId_2": {
          "name": "Jane Doe",
          "email": "jane@doe.com",
          "role": 1
        }
      }
    }
  }
}
```

and here are some test cases

```javascript,linenums=true
it('should allow simplelogin:2 (member) to create todolist for himself', function(done) {

    var ref = new Firebase(FB_BASE_URL);

    ref.authWithPassword({
      email: TEST2_ACCOUNT.email,
      password: TEST2_ACCOUNT.pwd
    }, function(error, authData) {

      var todoRef = ref.child("organizations").child("-uniqueOrgId_1").child("todos").child("uniqueStaffId_2");

      todoRef.push({
        title: 'My first to do'
      }, function(error) {
        should.not.exist(error);
        done();
      });

    });

  });

  it('should allow simplelogin:1 (admin) to create todolist for simplelogin:2 (member)', function(done) {

    var ref = new Firebase(FB_BASE_URL);

    ref.authWithPassword({
      email: TEST1_ACCOUNT.email,
      password: TEST1_ACCOUNT.pwd
    }, function(error, authData) {

      var todoRef = ref.child("organizations").child("-uniqueOrgId_1").child("todos").child("uniqueStaffId_2");

      todoRef.push({
        title: 'A todo created by admin for member'
      }, function(error) {
        should.not.exist(error);
        done();
      });

    });

  });

  it('should allow simplelogin:1 (admin) to read todolist of simplelogin:2 (member)', function(done) {

    // we need to first create the todo list for simple login 2
    var ref = new Firebase(FB_BASE_URL);

    ref.authWithPassword({
      email: TEST2_ACCOUNT.email,
      password: TEST2_ACCOUNT.pwd
    }, function(error, authData) {

      var todoRef = ref.child("organizations").child("-uniqueOrgId_1").child("todos").child("uniqueStaffId_2");

      todoRef.push({
        title: 'My first to do'
      }, function(error) {
        should.not.exist(error);

        // now we would log as admin and try to read his list
        ref.authWithPassword({
          email: TEST1_ACCOUNT.email,
          password: TEST1_ACCOUNT.pwd
        }, function(error, authData) {

          var todoRef = ref.child("organizations").child("-uniqueOrgId_1").child("todos").child("uniqueStaffId_2");

          todoRef.once("value", function(snapShot) {
            should.exist(snapShot);
            done();
          }, function(error) {
            should.not.exist(error);
            done();
          });

        });

      });

    });

  });

  it('should not allow simplelogin:2 (member) to read todolist of other users', function(done) {

    // we need to first create the todo list for simple login 1
    var ref = new Firebase(FB_BASE_URL);

    ref.authWithPassword({
      email: TEST1_ACCOUNT.email,
      password: TEST1_ACCOUNT.pwd
    }, function(error, authData) {

      var todoRef = ref.child("organizations").child("-uniqueOrgId_1").child("todos").child("uniqueStaffId_1");

      todoRef.push({
        title: 'My first to do'
      }, function(error) {
        should.not.exist(error);

        // now we would log as admin and try to read his list
        ref.authWithPassword({
          email: TEST2_ACCOUNT.email,
          password: TEST2_ACCOUNT.pwd
        }, function(error, authData) {

          var todoRef = ref.child("organizations").child("-uniqueOrgId_1").child("todos").child("uniqueStaffId_1");

          todoRef.once("value", function(snapShot) {
            should.not.exist(snapShot);
            done();
          }, function(error) {
            should.exist(error);
            done();
          });

        });

      });

    });

  });
```

and finally when we run these test cases -

```bash
$ mocha test3.js
Stage3 Test Cases
    ✓ should allow simplelogin:2 (member) to create todolist for himself (1549ms)
    ✓ should allow simplelogin:1 (admin) to create todolist for simplelogin:2 (member) (671ms)
    ✓ should allow simplelogin:1 (admin) to read todolist of simplelogin:2 (member) (1496ms)
    ✓ should not allow simplelogin:2 (member) to read todolist of other users (1284ms)


  4 passing (7s)
```

### Stage 4

[https://github.com/ksachdeva/YATODO/tree/ninja-stage-4](https://github.com/ksachdeva/YATODO/tree/ninja-stage-4)


Finally, the time to talk about how to implement the required business logic on the server side to get our user's email verified. The email verification is typically done in 2 different ways:

    * Send an email to the user with a link to a web url. User is supposed to click on it and clicking on it will execute the necessary logic at server.
    
    * Send an email to the user with an invite code. User is supposed to enter this code in his client application (mobile, desktop, cli etc) and it will submit the code to the server
    
We are going to take the second approach for this guide as we would not be running a server with a public url which the user can go to.

Now we should think about when the invite code generation and email dispatch would occur ? Ideally it should happen the moment user's profile is created. In other words, we need a server side code that is listening for 'users/$userId/profile' node creation and the moment it is detected the server side code would generate a unique invite code and email it to the user. 

At the time of writing of this guide, in order to achieve above requirement we would need to run our own server that can listen for profile node creation. However, this requirement would go away in near future versions. We will discuss more about it in the next stage (Stage-5) where we would improve what we would build in this stage.

In order to run our server, I am going to use nodejs and it will essentially consists of 2 standalone nodejs based workers. Next sections describe the workers

#### Worker 1

This worker will listen for the profile node additions. We would make use of firebase "child_added" event to capture that. Please note that this event would be fired when we will start to listen on the users node even though the profile node was created earlier. This is good as we would able to capture the "child_added" events even for situations where our worker would have started after profile node creation.

```bash
# if you want to try my repo then
git checkout -b ninja-stage-4 origin/ninja-stage-4
cd YATODO\src\server\workers
npm install
touch .env
node worker1.js
```

```bash
# if you want to create the files here are some instructions
cd YATODO\src\server\workers
npm init
npm install firebase --save
npm install lodash --save
npm install node-env-file --save
touch .env
touch fbutils.js
touch worker1.js
```

The .env file must contain the *FB_NAME* and *FB_SECRET_KEY* properties as we did in earlier stages. You have to do it even when you are using my repo as I do not submit my credentials to the source control :)

```javascript
// fbutils.js 
var env = require('node-env-file');

env(__dirname + '/.env');

var FB_NAME = process.env.FB_NAME;

var baseUrl = 'https://' + FB_NAME + "/";

exports.BASE_URL = baseUrl;
exports.FB_SECRET_KEY = process.env.FB_SECRET_KEY;
```

Now the worker1.js code :

```javascript,linenums=true
var Firebase = require('Firebase');
var fbUtils = require('./fbutils.js');
var _ = require('lodash');

var ref = new Firebase(fbUtils.BASE_URL);

ref.authWithCustomToken(fbUtils.FB_SECRET_KEY, function(error, authData) {

  if (error !== null) {
    console.log("failed to authenticate ..exiting");
    process.exit();
    return;
  }

  // start to monitor the users node for the addition of the nodes
  var usersRef = ref.child("users");

  usersRef.on("child_added", function(snapshot) {

    // send an email with the invite code
    var value = snapshot.val();
    var key = snapshot.key();

    var user_email = value.email;

    if (!_.has(value, 'verified')) {

      var generatedcode = "abcdefg"; // TODO : a random code with letters and digits only

      console.log("Generated Code - " + generatedcode);

      // push it in the users area
      var codeRef = usersRef.child(key).child('generated_code');
      codeRef.set(generatedcode);
    }

  });

});
```

Since we will always get 'child_added' events, we would only want to process the ones that do not have the 'verified' property. Once we have those nodes then we are generating a code ('abcdedfg' in this case but you are getting the drift of it) and emailing it (again a TODO for YATODO itself :)). And finally we are setting the 'generated_code' data structure in "users/$userId". 

```bash
# running our worker
node worker1.js
```

Good stuff but how do we test our little worker ? We now need a client application that would register the user and create the profile node. In order to address that I have created a small nodejs based cli application that is able to register a new user. The description of that is outside the scope of this guide however the code is available in the repository and you can use below instructions to run the client application.

```bash
# running the cli 
git checkout -b ninja-stage-4 origin/ninja-stage-4
cd YATODO\src\client\cli
npm install
touch .env
```

In .env file you would only need to set *FB_NAME*. This is a client application so it can not have our FB_SECRET_KEY !!!

```bash
# issue a command to register a new user
cd YATODO\src\client\cli
node yatodo-cli.js reg <email> <password>
```

If you manage to correctly and successfully run the cli, the worker1.js will receive a node added event and will do its job of creating the invite code and emailing it.

#### Worker 2

Worker 1 has done its job. Assume that user has recieved an email with an invite code. Now we need to provide support for user to submit this invitation code and a business logic on the server side that would verify the code. Remember we have stored the code at 'users/$userId/generated_code'.

In order to address above requirment we would need to introduce a new data structure that the user can write and our worker can listen to.

```javascript
// security rules
// I had added it in test4_rules.json file.
// you would want to paste its content to your firebase

"email_verification_submissions": {
      "$userId": {
        "code": {
          ".read": "(auth != null) && ($userId === auth.uid)",
          ".write": "(auth != null) && ($userId === auth.uid)"
        }
      }
    },

```

As clear from the above security rule, a user can read/write to only his specific node.

Now for the worker code

```javascript,linenums=true
// worker2.js
var Firebase = require('firebase');
var fbUtils = require('./fbutils.js');
var _ = require('lodash');

var ref = new Firebase(fbUtils.BASE_URL);

var submittedCodesCache = [];

ref.authWithCustomToken(fbUtils.FB_SECRET_KEY, function(error, authData) {

  if (error !== null) {
    console.log("failed to authenticate ..exiting");
    process.exit();
    return;
  }

  // start to monitor email_verification_submissions node

  var submissionsRef = ref.child("email_verification_submissions");

  submissionsRef.on("child_added", function(snapshot) {

    // send an email with the invite code
    var value = snapshot.val();
    var key = snapshot.key();

    var userRef = ref.child("users").child(key);
    userRef.child('generated_code').once('value', function(snapshot) {

      if (value.code === snapshot.val()) {
        // the code matched so we will set the verified
        // value to true
        console.log("submitted value matched the generated value ..");
        userRef.child('verified').set(true);
      } else {
        console.log("submitted and generated code did not match");
      }

      // in both cases remove the entry for submissions
      submissionsRef.child(key).remove();
      // and users generated_code
      userRef.child('generated_code').remove();

    }, function(error) {
      // ignoring the error here but we need to find a way
      // to inform the user that the code is invalid
      console.log("error occurred when getting the generated_code value");
    });

  });

});
```

If you follow the comments in the above code (worker2.js) you would see that now we are listening on 'email_verification_submissions' node. Once we receive an event we match the value of the submitted code with the generated code (stored in users/$userId). *(This is far from perfect, ideally we would want to set some status when the code does not match so that this information can be sent to the client)*.

```bash
# running our worker
node worker2.js
```

On the client side, I have the support to do verification as well:

```bash
# issue a command to register a new user
cd YATODO\src\client\cli
node yatodo-cli.js verify <email> <password> <code>
```

If you would provide the correct arguments for verify command then the worker2.js would kick in and set the 'verified' data structure to true and voila we have our user's email verified.

### Stage 5

[https://github.com/ksachdeva/YATODO/tree/ninja-stage-5](https://github.com/ksachdeva/YATODO/tree/ninja-stage-5)


Thanks to firebase infrastructure we have been able to find a storage for our data (user management, org management, todo items etc) & have support for mult-tenancy and security using the security rules language. We did not have to worry about the scalability of our solution until we introduced our servers (worker1 & worker2). Now we are responsible for making sure that they are capable of addressing heavy load. Wouldn't it have been great if all we were required was to write a simple server side business logic but actual running and scaling of the execution was not our problem !!

Unfortunately at the time of writing of this guide we can not achieve this desire completely but we can prepare for it. We would prepare for it by first simplifying our workers by trying not to run the computational intensive operations. Generating the invite code, preparing and sending email would be considered cpu heavy operations. 

But if we would not do invite code generation and email dispatch in our worker1.js where would we do ? Could we outsource this part to a cloud service ?. Fortunately the answer is YES.

We would make use of [iron.io's](http://iron.io) IronWorker technology to achieve our objective. IronWorker is a great innovation. In simple words, they allow you to submit your software program (written in java, ruby, python, nodejs etc) and run them in cloud. Note that you do not have to worry about manging the server. Behind the scenes they are dynamically provisioning the docker containers ( I suppose !!) on AWS (Amazon Web Services) and running your program in the environment specified by you.

[iron.io](http://iron.io) has a free tier that is more than sufficient for the needs of our guide. Please go to their website to register for a free account and create a new project. Every project has its own credentials (project_id and token). Once your project is created you should be able to download corresponding iron.json file. This file will contain the token and project_id and since it contains the credentials you would not find it my repo :)

You would want to install [Ironio's tool](http://dev.iron.io/worker/reference/cli/) to simplify code upload and queuing.


```bash
# if you would like to use my repo then do this
git checkout -b ninja-stage-5 origin/ninja-stage-5
cd YATODO\src\server\workers
npm install
touch .env
touch iron.json
```

```bash
# if you want to write the code yourself
# you should build upon the code from stage 4
cd YATODO\src\server\workers
npm install iron_worker --save
touch code_generator.worker
touch iw_code_generator.js
touch .env
touch iron.json
```

The code_generator.worker file describes the environment and necessary files

```
runtime "node"
stack "node-0.10"
exec "iw_code_generator.js"
dir "node_modules"
file "fbutils.js"
file "package.json"
file ".env"
```

and now the contents of iw_code_generator.js. You would see that it contains the code that we have extracted from worker1.js

```javascript
// content of iw_code_generator.js
var Firebase = require('firebase');
var fbUtils = require('./fbutils.js');
var _ = require('lodash');
var iron_worker = require('iron_worker');

var payload = iron_worker.params();

var ref = new Firebase(fbUtils.BASE_URL);

ref.authWithCustomToken(fbUtils.FB_SECRET_KEY, function(error, authData) {

  if (error !== null) {
    console.log("failed to authenticate ..exiting");
    process.exit();
    return;
  }

  // start to monitor the users node for the addition of the nodes
  var userRef = ref.child("users").child(payload.uid);

  // TODO : a random code with letters and digits only
  var generatedcode = "abcdefg";

  console.log("Generated Code - " + generatedcode);

  // push it in the users area
  var codeRef = userRef.child('generated_code');
  codeRef.set(generatedcode, function(error) {
    if (error) {
      console.log("error setting the code ..");
    }
    else{
        // TODO: email it
    }

    process.exit();

  });

});
```

The first thing to do is to submit our worker code to iron.io. Here is how you do it :

```bash
iron_worker upload code_generator.worker
```

If all goes well, then you should be able to see the code in your dashboard at iron.io. Also they do a fantastic job at versioning your worker code. If you modify the code and issue the above command then you would be submitting a new version.

Now that iron.io has our worker code we need to find a way to trigger its execution. This is called queuing a task. The iron_worker cli provides a command that you could use, you could also use the dashboard but none of them would not address our requirements. We are supposed to queue the task (or execute the worker) when we detect the users/$userid data structure creation. Let's look at our worker1.js file to see how we achieve it :

```javascript,linenums=true
var Firebase = require('firebase');
var fbUtils = require('./fbutils.js');
var _ = require('lodash');
var iron_worker = require('iron_worker');

var iw = new iron_worker.Client();

var ref = new Firebase(fbUtils.BASE_URL);

ref.authWithCustomToken(fbUtils.FB_SECRET_KEY, function(error, authData) {

  if (error !== null) {
    console.log("failed to authenticate ..exiting");
    process.exit();
    return;
  }

  // start to monitor the users node for the addition of the nodes
  var usersRef = ref.child("users");

  usersRef.on("child_added", function(snapshot) {

    // send an email with the invite code
    var value = snapshot.val();
    var key = snapshot.key();

    var user_email = value.email;

    if (!_.has(value, 'verified') &&
      !_.has(value, 'generated_code')) {

      // queue a worker
      var payload = {
        uid: key
      };
      var options = {
        priority: 1
      };

      console.log("Queuing a task to generate code and email it ..");
      iw.tasksCreate('code_generator', payload, options,
        function(error, body) {
          if (error) {
            console.log("An error occurred when queuing a task ..");
          }
        });
    }

  });

});
```

Essentially we are making use of iron_worker node library that is not only helping us queue the task  but also provides the spport to pass a payload to task. The payload in this case is the uid of the user for whom we are supposed to generate the code.

Similar things can be done for worker2.js such that it only listens for 'child_added' event on 'email_verification_submissions' data structure and delegate the cpu intensive workload to an IronWorker based task.

Firebase is working on a new feature called "Triggers". The triggers would get similar treatment as that of security rules. You would be able to define some action (most likely outgoing http requests) when certain conditions are met. Thanks to that you would be able to get rid of worker1.js and worker2.js by queuing the ironworker tasks directly from the firebase backend.

### Stage 6 (Next Steps)

This was a relatively longer article/guide yet we did not manage to cover all aspects of technologies we touched. 

We are left with some tasks to be completed for our solution. The main one would be 

- Build the server side logic for creating the organization node and register the creator as the admin

I am leaving that as an exercise for the readers of this article (*Hint that it is not very different from user's email verification* :))

If you wish you can check the solution, it is in the stage-6 branch but I would insist to first try on your own as it would be a great exercise.

You would also find in the stage-6 branch an [ionic](http://ionicframework.com) application that makes use of our firebase backend. 

Describing the ionic application is outside the scope of this guide and you would find great number of articles that describe how to build an ionic application that connects to firebase.

```bash
git checkout -b ninja-stage-6 origin/ninja-stage-6
cd YATODO\src\client\mobile
npm install
bower install
ionic serve --lab
```

[https://github.com/ksachdeva/YATODO/tree/ninja-stage-6](https://github.com/ksachdeva/YATODO/tree/ninja-stage-6)


## Conclusion

It is amazing to see how easy it has become to build, deploy and scale a modern web application/service. Thanks to the innovations of firebase, iron.io and ionic and many other cloud & mobile platforms like them we as developers get to focus more on the user experience and the core problem that we want to solve.
















































