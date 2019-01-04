Proposal: Support Robot account in Harbor

Author: Yan Wang

## Abstract

Robot account is a machine user that have the permission to access harbor resources, like pull/push docker images. It's a better way to integrate harbor into your CI/CD workflow without using a user account, especially in LDAP mode.

## Motivation

There are a lot of requirements from community would like to have the ability to work harbor with robot account. But, Harbor only has the capability of using the registered user to access resources, and do not support user to pull image with robot acccout.

## User Stories

### Story 1
As a project administrator, I want to be able to create a robot account and grant project read/write permission to it.

### Story 2
As a dev ops developer, I want to be able to use the token of robot account to pull image.

## Solution

This proposal only targets on the pull/push images workflow, and the design includes DB, API, authn/authz.

### DB scheme

1, the robot account is not a harbor user, so an new table introduced to store the robot info.

```yaml

create table harbor_robot (
 robot_id SERIAL PRIMARY KEY NOT NULL,
 name varchar(255),
 token varchar(255),
 project_id int NOT NULL,
 desc varchar(255),
 deleted boolean DEFAULT false NOT NULL,
 creation_time timestamp default CURRENT_TIMESTAMP,
 update_time timestamp default CURRENT_TIMESTAMP,
 UNIQUE (name)
);

```

### API

The project admin could manager the robot accounts of the project, all the api are project admin only. 

1, Adding a robot account

````

POST /api/robots
Data: 
{
  name: "test1",
  desc: "test1_desc",
  // currently, it always set to p as the robot account is the project level.
  scope: "p",
  project_id: 1,
  access: [
    {"name":"project/repo", "actions":["read"]},
    {"name":"project/label", "actions":["write"]}
  ]
}

````

2, Deleting a robot account

````

DELETE /api/robots/${id}         

````

3, View a/all robot account

````

GET /api/robots
GET /api/robots/${id}

````

4, Update a robot account

````

PUT /api/robots/${id}
Data: 
{
  desc: "test1_desc",
}

````

5, Update the access of a robot account

````

PUT /api/robots/${id}
Data: 
{
  access: [
      {"name":"project/repo", "actions":["write"]}
  ]
}

````

It's actually to issue an new token bases on the updates in the background.

## Login 

### UI login
Not supported

````
The robot account cannot login to harbor portal.

No code change as all the robot accounts are stored in harbor_robots instead of harbor_user.
````

### Docker login

To distinguish the robot account from user, it will add a predefined prefix "harborobots" to the name of robot, like "harborobots_example1".

1, with user/pwd

````

docker login harbor.example.com
Username: harborobots_robotexample
Password: rKgjKEMpMEK23zqejkWn5GIVvgJps1vKACTa6tnGXXyOlOTsXFESccDvgaJx047q

````

2, with config.json

````

{
  "auths": {
    “harbor.example.com”: {
      "auth": "Zmcm01Szl2eXc0elJOU2pkaEgvR1YrUjdCUXFIeWtQMTFkWWZXSUV0YU13cWhcbnllZjR1K2dUQytrYk81R002eUhqcmJFUGxHcW03WDU4UWtxd2JDbTdhMllnNi9SM2hl",
    }
  }
}

````

### AuthN/AuthZ

- The authn for robot account rely on the DB.

#### Modify the db authenticate to support robots login.
The current login on db_auth is hard to extend to support robots as it's user binding, so the easiest way to handle this is to add a new login func(LoginRobot) for robots.
https://github.com/goharbor/harbor/blob/master/src/core/auth/authenticator.go#L130

```go

func LoginRobot(m models.AuthModel) (*models.Robot, error) {

	authMode, err := config.AuthMode()
	if err != nil {
		return nil, err
	}
	if authMode == "" || dao.IsSuperUser(m.Principal) {
		authMode = common.DBAuth
	}

	authenticator, ok := registry[authMode+"_robot"]
	log.Debug("Current AUTH_MODE is ", authMode)
	if !ok {
		return nil, fmt.Errorf("Unrecognized auth_mode: %s", authMode)
	}
	if lock.IsLocked(m.Principal) {
		log.Debugf("%s is locked due to login failure, login failed", m.Principal)
		return nil, nil
	}
	robot, err := authenticator.Authenticate(m)
	if err != nil {
		if _, ok = err.(ErrAuth); ok {
			log.Debugf("Login failed, locking %s, and sleep for %v", m.Principal, frozenTime)
			lock.Lock(m.Principal)
			time.Sleep(frozenTime)
		}
		return nil, err
	}
	err = authenticator.PostAuthenticate(user)
	return robot, err
}

```

#### Add a new context modifier as below to handle the auth of robot account.

````

secretReqCtxModifier => robotsReqCtxModifier => basicAuthReqCtxModifier =>  sessionReqCtxModifier => unauthorizedReqCtxModifier

robotsReqCtxModifier --- robot login -- DB (harbor_robots)

````

```go

type robotAuthReqCtxModifier struct{}

func (b *robotAuthReqCtxModifier) Modify(ctx *beegoctx.Context) bool {
	username, password, ok := ctx.Request.BasicAuth()
	if !ok {
		return false
	}
	if !strings.HasPrefix("username", "harborobots") {
		return false
	}
	log.Debug("got user information via basic auth")

	user, err := auth.LoginRobot(models.AuthModel{
		Principal: username,
		Password:  password,
	})	
	if err != nil {
		log.Errorf("failed to authenticate %s: %v", username, err)
		return false
	}
	if user == nil {
		log.Debug("basic auth user is nil")
		return false
	}
	log.Debug("using local database project manager")
	pm := config.GlobalProjectMgr
	log.Debug("creating local database security context...")
	securCtx := local.NewSecurityContext(user, pm)
	setSecurCtxAndPM(ctx.Request, securCtx, pm)
	return true
}


```

- The authz for robot account rely on the jwt token.

#### Token

##### Config items

| Item               | Value          | Level  |
| ------------------ | -------------- | ------ |
| Robot_Token_Expire | 30             | User   |
| Robot_Token_Key    | /etc/robot_key | System |

##### Model

```go

type Token struct {
    Raw       string                 
    Method    SigningMethod          
    Header    map[string]interface{} 
    Claims    Claims                 
    Signature string                 
    Valid     bool                   
}

type Claims struct {
    Audience  string `json:"aud,omitempty"`
    ExpiresAt int64  `json:"exp,omitempty"`
    Id        string `json:"jti,omitempty"`
    IssuedAt  int64  `json:"iat,omitempty"`
    Issuer    string `json:"iss,omitempty"`
    NotBefore int64  `json:"nbf,omitempty"`
    Subject   string `json:"sub,omitempty"`
    Access []*ResourceActions `json:"access"`
}

type ResourceActions struct {
	Name    string   `json:"name"`
	Actions []string `json:"actions"`
}

```

##### Scpoe -- Implement a new security context for robot account

In the implementation of robot context, the func of HasReadPerm and HasWritePerm could validate the token scope, and block all the request to harbor with forbidden. 

```go

./src/common/security/robots/context.go

// SecurityContext implements security.Context interface based on database
type SecurityContext struct {
	robots *models.Robots
	pm   promgr.ProjectManager
}

// NewSecurityContext ...
func NewSecurityContext(robots *models.Robots, pm promgr.ProjectManager) *SecurityContext {
	return &SecurityContext{
		robots: robots,
		pm:   pm,
	}
}

// IsAuthenticated returns true if the user has been authenticated
func (s *SecurityContext) IsAuthenticated() bool {
	return s.robots != nil
}

// GetUsername returns the name of the authenticated robot account
// It returns null if the robot account has not been authenticated
func (s *SecurityContext) GetUsername() string {
	if !s.IsAuthenticated() {
		return ""
	}
	return s.robots.name
}

// IsSysAdmin ...
func (s *SecurityContext) IsSysAdmin() bool {
	return false
}

// IsSolutionUser ...
func (s *SecurityContext) IsSolutionUser() bool {
	return false
}

// HasReadPerm returns whether the user has read permission to the project
func (s *SecurityContext) HasReadPerm(projectIDOrName interface{}) bool {
	// private project
    if !s.IsAuthenticated() {
        return false
    }
    
    robots := s.GetProjectRobots(projectIDOrName)
    if len(roles) > 0 {
        // needs to decode the token to get the access	
    }
}

// HasWritePerm returns whether the user has write permission to the project
func (s *SecurityContext) HasWritePerm(projectIDOrName interface{}) bool {
    // private project
    if !s.IsAuthenticated() {
        return false
    }
    
    robots := s.GetProjectRobots(projectIDOrName)
    if len(roles) > 0 {
        // needs to decode the token to get the access	
    }
}

// HasAllPerm returns whether the user has all permissions to the project
func (s *SecurityContext) HasAllPerm(projectIDOrName interface{}) bool {
	return false
}

// GetProjectRobots ...
func (s *SecurityContext) GetProjectRobots(projectIDOrName interface{}) []int {
	return nil
}

// GetMyProjects ...
func (s *SecurityContext) GetMyProjects() ([]*models.Project, error) {
	result, err := s.pm.List(
		&models.ProjectQueryParam{
			Member: &models.MemberQuery{
				Name:      s.GetUsername(),
				GroupList: s.user.GroupList,
			},
		})
	if err != nil {
		return nil, err
	}

	return result.Projects, nil
}


```