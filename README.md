# FM-HAM
### Headless Authorization Module for FileMaker

FM-HAM is a flexible non opinionated model for managing account authorization. Note: Authorization is what the user can do vs authentication - who the user is.

This module is headless meaning it does not come with a user interface and often one is not needed. Technically the module is also foot / tailless as well. This means that you can store the raw authorizations JSON objects any way you want but typically they are saved in a single field within your user or entity record.

## Features
- Non-opinionated
- Ability to track numerical, value-based, or boolean priv's
- permissions can be stacked and overridden
- No limit the depth of permissions
- easily pass a permissions object to 3rd party logic such as web interfaces like www.fmBetterForms.com etc.
- Use it anywhere, including FileMaker record level access

## Use Cases
FM-HAM Can be used anywhere you need to control user activities and quantities
- Control number of things a user is allowed to create
- Control access a user has to an area of an application
- Multi-Tenant verticals, makes managing account attributes a snap

## Simple Example ## 

# Overview
Define groups for the privledges within your application. Here is a simple example:

```
{
	"admin": {
		"inherit": ["manager"],
		"addUsers": true
	},
	"manager: {
		"inherit": ["basic"],
		"editUsers": true,
		"sessionLimit": 3
	},
	"basic": {
		"addUsers": false,
		"editUsers": false,
		"editOwnDetails": true,
		"sessionLimit": 1
	}
}
```

To nest privleges, use the `inherit` keyword. Then, you'll only need to define or re-define privledges that are different than what the group is inheriting from.

- `inherit` must be an array of strings that match other group names
- Inheritance applies in order of the array; groups later in the array will take precedence.
- Inheritance uses recursion; be careful not to cause an infinite loop!

Each will then have their own user object with the group they belong to, and any user-specefic overrides

```
{
	"group": "manager",
	"overrides": {
		"sessionLimit": 5
	}
}
```

## Usage

Call the `HAM_Config()` function once per session to setup the privileges for the current user. If we passed in the groups and user definitions above, it would produce the following flattened session object:

```
{
	"addUsers": false,
	"editUsers": true,
	"editOwnDetails": true,
	"sessionLimit": 5
}
```

Then, you can use the `HAM_CheckPriv()` function to lookup any of these privledges before performing certain actions.

Example:
```
HAM_CheckPriv( "addUsers" ; "" ) // will return FALSE
```

- the `HAM_CheckPriv` function attempts to return the value at the key provided. If it encounters any errors, it will return `False` by default. So name your pr


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

- `disabled`
- `accountDisabled`
- `widgets`
- `preventGlobalThermonuclarWar`

**Good Names**

- `isActive`
- `isDisabled`
- `canAccessThis`
- `countWidgets`

Be sure to make your permissions as permissive as possible. If the `CheckPriv` function fails, it will return `False` by default.


# Evaulations

If you add `_eval` to the end of any privledge name, the result of that privledge will be evalutated using FileMaker's `Evaluate()` function whenever it's checked by the `HAM_CheckPriv()` function. That means you can take advantage of local variables or even SQL Lookups within those calculations.

In this simple example, the `editRooms` priviledge will only be true on Mondays.
```
"editRooms_eval": "Get ( DayOfWeek ) = 2",
```

# Options & Defaults

Various options can be passed to the `Setup` and `CheckPriv` functions to modify the behavior of HAM. The options are passed as a JSON object as the last parameter of these functions.

### `HAM_Config` Options

**evaluateNow** (boolean, defaults `False`) - Run all of the `_eval` privildeges during setup instead of each time the privledge is checked. This is helpful if you have a lot of calulations or if they take a long time to run.

**storeGlobal** (boolean, defaults `True`) - If this is set to `False`, the $$HAM_Config global variable will never be stored in FileMaker's memory. This can make HAM more secure, but also may run more slowly since the Setup will be run each time `HAM_CheckPriv()` is called.

### `HAM_CheckPriv` Options

**returnError** (boolean, defaults `False`) - If true, the function will return a standard error object if an error occured (instead of the default `False`). You can also always check the `$$HAM_LastError` variable to see if any errors occcured. With this enabled, if no errors occur, the value from the flat object of privileges is still returned.

### Default Configuration

You can edit the default configuration for the `HAM_Config` function. These settings will act like the normal parameters of the function if the functino is run with blank parameters, or if the CheckPriv function runs befor the Config function.


#### Script example psudo code
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
