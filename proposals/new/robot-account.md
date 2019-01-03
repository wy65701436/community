Proposal: Enable Robot accounts in Harbor

Author: Yan Wang

## Abstract

Robot account is a machine user that have the permission to access harbor resouces, like pull/push docker images. It's a better way to integreate harbor into your CI/CD workflow without using a user account, especially in LDAP mode.

## Motivation

Currently, Harbor only has the capability of using the registered user to access resources, and do not support user to pull image with token.

## Solution

This proposal only targets on the pull/push images workflow, includs DB, API, authn/authz.

### DB scheme/model

1, the robot account is not a harbor user, so an new table introduced to store the accounts info.

```yaml

create table harbor_robot (
 robot_id SERIAL PRIMARY KEY NOT NULL,
 name varchar(255),
 token varchar(255),
 expire varchar(40) NOT NULL,
 project_id int NOT NULL,
 desc varchar(255),
 deleted boolean DEFAULT false NOT NULL,
 creation_time timestamp default CURRENT_TIMESTAMP,
 update_time timestamp default CURRENT_TIMESTAMP,
 UNIQUE (name),
 UNIQUE (token)
);

```

2, the permission of robot accounts

```yaml

create table harbor_robot_permission (
  id SERIAL PRIMARY KEY NOT NULL,
  robot_id int NOT NULL,
  project_id int NOT NULL,
  /*
   R/W
  */
  scope char(1)
)

```

### API -- managing robot account

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
  project_id: 1
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

### API -- setting permission

1, Adding/Updating permission for a robot account 

````

PUT /api/robots/${id}/permission
Data:
{
  project_id: 2,
  /*
   R/W
   R: read
   W: write
  */
  role: R
}

````

2, Deleting permission for a robot account

````

DELETE /api/robots/${id}/permission

````

## Login 

### UI login
Not supported

````
The robot account cannot login to harbor portal.

No code change as all the robot accounts are stored in harbor_robots instead of harbor_user.
````

### docker login

1, with user/pwd

````

docker login harbor.example.com
Username: harborrobots_robotexample
Password: rKgjKEMpMEK23zqejkWn5GIVvgJps1vKACTa6tnGXXyOlOTsXFESccDvgaJx047q

````

2, with config.json

````

{
  "auths": {
    “harbor.example.com”: {
      "auth": "rKgjKEMpMEK23zqejkWn5GIVvgJps1vKACTa6tnGXXyOlOTsXFESccDvgaJx047q",
    }
  }
}

````

### docker login authn

To distinguish the robot account from user, it will add a predefined prefix "harborrobots" to the name of harbor robots, like "harborrobots_example1".

#### Modify the db Authenticate to support robots login.
The current login on db_auth is hard to extend to support robots as it's user binding, so the easist way to handle this is to add a new login func for robots.
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
	log.Debug("Current AUTH_MODE is ", authMode)

	authenticator, ok := registry[authMode+"_robot"]
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

secretReqCtxModifier => robotsReqCtxModifier => basicAuthReqCtxModifier =>  unauthorizedReqCtxModifier
robotsReqCtxModifier --- robot auth -- DB (harbor_robots)

````

```go

type robotAuthReqCtxModifier struct{}

func (b *robotAuthReqCtxModifier) Modify(ctx *beegoctx.Context) bool {
	username, password, ok := ctx.Request.BasicAuth()
	if !ok {
		return false
	}
        if !strings.HasPrefix("username", "harborrobots") {
		return false
	}
	log.Debug("got user information via basic auth")

	user, err := auth.LoginRobot(models.Robots{
		Name: username,
		Token:  password,
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


#### Implement a new security context for robot account

./src/common/security/robots/context.go

```go

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
	// public project
	public, err := s.pm.IsPublic(projectIDOrName)
	if err != nil {
		log.Errorf("failed to check the public of project %v: %v",
			projectIDOrName, err)
		return false
	}
	if public {
		return true
	}

	// private project
	if !s.IsAuthenticated() {
		return false
	}

	robots := s.GetProjectRobots(projectIDOrName)
	return len(robots) > 0
}

// HasWritePerm returns whether the user has write permission to the project
func (s *SecurityContext) HasWritePerm(projectIDOrName interface{}) bool {
	if !s.IsAuthenticated() {
		return false
	}

	robots := s.GetProjectRobots(projectIDOrName)
	for _, robot := range robots {
		switch role.permission {
		case common.PermissionWrite:
			return true
		}
	}
	return false
}

// HasAllPerm returns whether the user has all permissions to the project
func (s *SecurityContext) HasAllPerm(projectIDOrName interface{}) bool {
	return false
}

// GetProjectRobots ...
func (s *SecurityContext) GetProjectRobots(projectIDOrName interface{}) []int {
	return nil
}

// GetPermissionsByRobot - Get the group role of current user to the project
func (s *SecurityContext) GetRolesByGroup(projectIDOrName interface{}) []int {
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
