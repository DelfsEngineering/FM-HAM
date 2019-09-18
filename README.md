# FM-HAM
### Headless Authorization Module for FileMaker

FM-HAM is a flexible non opinionated model for managing account authorization. Note: Authorization is what the user can do vs authentication - who the user is.

This module is headless meaning it does not come with a user interface and often one is not needed. Technically the module is also foot / tailless as well. This means that you can store the raw authorizations JSON objects any way you want but typically they are saved in a single field within your user or entity record.

## Features
- Non-opinionated
- Ability to track numerical as well as boolean priv's
- permissions can be stacked and overridden
- No limit the depth of permissions
- easily pass a permissions object to 3rd party logic such as web interfaces like www.fmBetterForms.com etc.

## Use Cases
FM-HAM Can be used anywhere you need to control user activities and quantities
- Contorl number of things a user is allowed to create
- Control access a user has to an area of an application

## Simple Example ## 

# Overview
Each user / entity has a permissions JSON object stored with their corresponding record.

## Details

```
// example
{
	"viewAllReservations": true,
	"viewOwnReservations": true,
	"editAllReservations": false,
	"editOwnReservations": true,
	"editEquipment": false,
	"editRooms_eval": "Get ( DayOfWeek ) = 2", # evalutated every time editRooms is checked
	"editAnnoucments": false,
	"editOtherUsers": false,
	"showAdminMenu": true,
	"...": "..."
}
```

## Best Practice
Naming permissions and privs should be as symnatic as poosible. This will allow you and other developers to easily make assumptions as to waht a prive key mean.

Boolean keys should be more truthy than falsey eg 

**Bad names**

`disabled`
`accountDisabled`
`widgets`

**Good Names**

`isActive`
`isDisabled`
`canAccessThis`
`countWidgets`

** TODO - Think we may want a math based system for numeric values
eg: 
`sumWigets` would add all the sub privs together
`maxAttempts` would take the maximum of all subs
etc


#### Concepts

Using a custom function in FM, we can check the boolean, or mumeric aggrigate value of any of these keys before running a script.

I think that CF should just return a boolean value or a number, errors should be raised manually, but another CF could easily be written to raise a generic “insufficient privileges” error code.


++How to configure that CF to work with any user table??++

#### Special Keys
`inherit` - array will include the listed group names privs in the specified order

The permission object is based on groups with predefined attributes
{
	"admin": {
		"inherit": ["cage_manager"],
		"showAdminMenu": true,
		"notifyManagerUponApprove": false,
		"notifyManagerUponUnlock": false,
	},
	"cage_manager": {
		"inherit": ["student"],
		"viewReservations": "all",
		"editReservations": "all",
		"inOut": true,
		"editReservations": "always",
		"approveReservations": true,
		"notifyManagerUponApprove": true,
		"unlockReservations": true,
		"notifyManagerUponUnlock": true,
		"newWidget1": true
	},
	"student": {
		"viewReservations": "own",
		"editReservations": "own",
		"inOut": false,
		"approveReservations": false,
		"notifyManagerUponApprove": false,
		"unlockReservations": false,
		"notifyManagerUponUnlock": false,
		"showAdminMenu": false,
	},
	"...": {}
}

​
//Priv Specification
{
	"editAllReservations": {
		"own": "You can only edit your own reservations",
		"all": "You can't edit all reservations"
	}
}

```
Global script populates a `priv` object with the user’s permissions object
If [ CheckPriv ( "editReservations" ; "" ) ] #return boolean
	Set Variable [ $$BF_Actions ; BF_SetAction (show alert = $priv_error) ]
	Exit Loop If [ True ]
End If
```



```
If [ not CheckPriv ( "editReservations" ) = "all" ]
	Set Variable [ $$BF_Actions ; BF_SetAction (show alert = $priv_error) ]
	Exit Loop If [ True ]
Else If [ not CheckPriv ( "editReservations" ) = "own" ]
	CheckPrivError ( "editReservations" ; "own" )
End If
​
Exit Loop If [ CheckPriv ( "showAdminMenu" ; "" ) ]

```

### Misc Notes to be converted into code

A user can also have custom privileges that can override these default group settings. A FileMaker Custom Function can merge the custom privileges with the group privileges into the user’s JSON object, with the custom settings taking priority
++How to verify the default setting for privileges if not set?++

++How to manage UI for granting/removing privileges so that that when changing privileges leftover custom privileges aren’t ignored?++

In FileMaker, we can easily make a series of tables to easily manage the groups and permission lists. This would allow easy access to add new groups or individual privileges. Scripts would normalize this data into JSON for a solution.
​

## Functions
`ham_carve (  )` setup
returns error object if problem
#### Always returns boolean
`CheckPriv ( priv )`
`CheckCount ( priv ; num )` num or simple string comparator, eg `"<=5"`
#### Always returns value (string)
`ReturnPriv ( priv )` 

### TODO's
- Add demo and integration with and self tests ( all in one)
- Define the functions needed from a mock example's requirements

