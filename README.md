# FM-HAM
Headless Authorization Module for FileMaker

# BetterForms/FileMaker JSON Authorization

Each user has a permissions JSON object stored with their user record in a user table
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

Using a custom function in FM, we can check the boolean value of any of these keys before running the script.
I think that CF should just return a boolean value, errors should be raised manually, but another CF could easily be written to raise a generic “insufficient privileges” error code.


++How to configure that CF to work with any user table??++

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

Global script populates a `priv` object with the user’s permissions object
If [ CheckPriv ( "editReservations" ; "" ) ] #return boolean
	Set Variable [ $$BF_Actions ; BF_SetAction (show alert = $priv_error) ]
	Exit Loop If [ True ]
End If

​
```
If [ not CheckPriv ( "editReservations" ) = "all" ]
	Set Variable [ $$BF_Actions ; BF_SetAction (show alert = $priv_error) ]
	Exit Loop If [ True ]
Else If [ not CheckPriv ( "editReservations" ) = "own" ]
	CheckPrivError ( "editReservations" ; "own" )
End If
```
​
Exit Loop If [ CheckPriv ( "showAdminMenu" ; "" ) ]
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