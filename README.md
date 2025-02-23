# FM-HAM

### Headless Authorization Module for FileMaker

FM-HAM is a flexible, non-opinionated model for managing authorization in FileMaker solutions. (Remember: **authorization** defines what a user can do, while **authentication** defines who the user is.) The module is “headless” – it does not provide a user interface. Instead, it produces raw JSON objects that you can store anywhere in your solution (typically in a field on a user or entity record) and pass to third-party logic.

---

## Overview & Use Cases

FM-HAM enables you to:

- Control what a user is allowed to do (e.g., the number of records they can create, which parts of the app they can access).
- Support numerical, boolean, and value-based privileges.
- Stack and override permissions via JSON merging.
- Define a complex hierarchy of privileges using inheritance.

**Example Groups Object:**

```json
{
  "basicUser": {
    "addUsers": false,
    "editUsers": false,
    "editOwnDetails": true,
    "widgetsAllowed": 1
  },
  "manager": {
    "inherit": [
      "basicUser"
    ],
    "editUsers": true,
    "widgetsAllowed": 3
  },
  "admin": {
    "inherit": [
      "manager"
    ],
    "addUsers": true
  }
}
```

- The **inherit** key is used to nest privileges. Inheritance is applied in order (later groups override earlier ones) and works recursively.
- Each user has their own JSON object that specifies their group and any custom overrides:

```json
{
  "group": "manager",
  "overrides": {
    "sessionLimit": 5
  }
}
```

---

## Detailed Function Descriptions

### 1. HAM\_Config

#### Purpose

Sets up the authorization privileges for the current user by merging group definitions and any custom overrides. It creates a flattened JSON object of all applicable privileges.

#### Parameters & Defaults

- **groups**: JSON object defining group privileges.
  - If empty, defaults are provided (via `default_groups`).
- **user**: JSON object defining the user's group and any custom privilege overrides.
  - If empty, defaults are provided (via `default_user`).
- **options**: JSON object for configuration options.\
  **Available Options:**
  - **evaluateNow** (boolean, default `False`)\
    If set to true, any privilege key ending in `_eval` is evaluated during setup (using FileMaker’s `Evaluate()` function) rather than at the time of privilege checking. This is useful when you have many or time-consuming calculations.
  - **storeGlobal** (boolean, default `True`)\
    Determines whether the resulting flattened JSON is stored in a global variable (`$$HAM_Config`). Setting it to false can improve security by avoiding persistent globals, but may slow performance if re-calculation is needed on each call.
  - **encryptionKey** (string, optional)\
    When provided, the flattened JSON object is encrypted. Every call to `HAM_CheckPriv` will need the same key for decryption.

#### Example Usage

```filemaker
Set Variable [ $groupsJSON; Value: "{ ... }" ]
Set Variable [ $userJSON; Value: "{ 'group': 'manager', 'overrides': { 'sessionLimit': 5 } }" ]
Set Variable [ $options; Value: JSONSetElement ( "{}" ; [ "evaluateNow"; True; JSONBoolean ], [ "storeGlobal"; True; JSONBoolean ] ) ]

Set Variable [ $config; Value: HAM_Config ( $groupsJSON ; $userJSON ; $options ) ]
```

If no errors occur, `$config` will contain a flattened JSON object similar to:

```json
{
  "addUsers": false,
  "editUsers": true,
  "editOwnDetails": true,
  "sessionLimit": 5,
  "widgetsAllowed": 3
}
```

---

### 2. HAM\_CheckPriv

#### Purpose

Retrieves the value of a specific privilege from the flattened configuration object. If the key contains an `_eval` suffix, the evaluation is performed (unless already pre-evaluated).

#### Parameters & Options

- **privName**: The name of the privilege to check.
- **options**: JSON object for configuration options.\
  **Available Options:**
  - **returnError** (boolean, default `False`)\
    If true, the function returns a standard error object when an error occurs instead of defaulting to `False`.
  - **encryptionKey** (string, optional)\
    Needed if the flattened configuration was encrypted.
  - **flat** (optional)\
    Allows direct passing of the flattened privileges object to bypass the global `$$HAM_Config`.

#### Example Usage

```filemaker
Set Variable [ $canAddUsers; Value: HAM_CheckPriv ( "addUsers" ; "{}" ) ]
If [ not $canAddUsers ]
  // Deny action or alert the user
End If
```

---

### 3. HAM\_Utility

#### Purpose

A multipurpose utility function used internally by both `HAM_Config` and `HAM_CheckPriv`. It handles tasks such as error generation, JSON validation, privilege merging, and evaluation.

#### Key Options

- **"error"**\
  Creates a standardized JSON error object using the provided error code, description, and additional data.
- **"isError"**\
  Checks whether a given JSON object represents an error.
- **"JSON.IsValid"**\
  Validates whether a JSON string is correctly formatted.
- **"\_buildPrivSet"** & **"buildPrivSet"**\
  Recursively builds a flattened privilege set by merging inherited privileges.
- **"objectAssign"**\
  Merges two JSON objects. Keys ending with `_max` or `_sum` are combined using the appropriate FileMaker functions (Max or Sum).
- **"stripMath"**\
  Processes privilege values that begin with a number but contain other characters.
- **"evaluateNow"**\
  Iterates over the flattened configuration to evaluate any keys ending in `_eval` and then updates the configuration accordingly.

---

## Best Practices for Using FM-HAM

1. **Semantic Naming:**\
   Use clear and descriptive names for privileges.
2. **Evaluation of Privileges:**\
   Append `_eval` to dynamically calculated privileges.
3. **Custom Overrides:**\
   Ensure user-specific values correctly override group defaults.
4. **Error Handling:**\
   Check for errors using `$$HAM_LastError` or `returnError`.
5. **Global vs. Session Configuration:**\
   Use `storeGlobal` for performance but encrypt if needed.

---

