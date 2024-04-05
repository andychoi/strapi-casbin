# Node RBAC Examples
This repository contains a number of Node.js applications, each demonstrates a different approach to implementing Resource-Based Authorization (RBAC) for a simple HTTP API.

refer: https://www.aserto.com/blog/building-rbac-in-node

## Running the Examples
To run an example, cd to its directory and run: 

```sh
npm install
npm start
```
## Testing

To test the application, you can make a set of requests to the routes and check the responses:

```
curl -X <HTTP Verb> --location 'http://localhost:8080/api/<asset>' \
--header 'Content-Type: application/json' \
--data-raw '{
    "user": {
        "id": "beth@the-smiths.com"
    }

}'
```

Where `<HTTP Verb>` is either `GET`, `PUT`, or `DELETE` and `<asset>` is either `megaSeeds` or `timeCrystals`.


## Node-Casbin
Casbin is a powerful and efficient open-source access control library. It has SDKs in many languages, including Javascript, Go, Rust, Python, and more. It provides support for enforcing authorization based on various access control models: from a classic "subject-object-action" model, through RBAC and ABAC models to fully customizable models. It has support for many adapters for policy storage.

In Casbin, the access control model is encapsulated in a configuration file (src/rbac_model.conf):
```
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[matchers]
m = g(r.sub , p.sub) && r.obj == p.obj && r.act == p.act

[policy_effect]
e = some(where (p.eft == allow))
```

Along with a policy/roles definition file (src/rbac_policy.conf)

```
p, clone, megaSeeds, gather
p, clone, timeCrystals, gather
p, sidekick, megaSeeds, consume
p, sidekick, timeCrystals, consume
p, evilGenius, megaSeeds, destroy
p, evilGenius, timeCrystals, destroy
g, sidekick, clone
g, evilGenius, sidekick
```

- The request_definition section defines the request parameters. In this case, the request parameters are the minimally required parameters: subject (sub), object (obj) and action (act). It defines the parameters' names and order that the policy matcher will use to match the request.
- The policy_definitions section dictates the structure of the policy. In our example, the structure matches that of the request, containing the subject, object, and action parameters. In the policy/roles definition file, we can see that there are policies (on lines beginning with p) for each role (clone, sidekick, and evilGenius)
- The role_definition sections is specific for the RBAC model. In our example, the model indicates that an inheritance group (g) is comprised of two members. In the policy/roles definition file, we can see two role inheritance rules for sidekick and evilGenius, where sidekick inherits from clone and evilGenius inherits from sidekick (which means the evilGenius will also have the clone permissions).
- The matchers sections defines the matching rules for policy and the request. In our example, the matcher is going to check whether each of the request parameters match the policy parameters, and that the role r.sub is in the policy.
The implementation of the hasPermission middleware function for Node-Casbin is as follows:

```
const hasPermission = (action) => {
  return async (req, res, next) => {
    const { user } = req.body;
    const { asset } = req.params;
    const userRoles = resolveUserRoles(user);

    const e = await newEnforcer("./rbac_model.conf", "./rbac_policy.csv");

    const allowed = await userRoles.reduce(async (perms, role) => {
      const acc = await perms;
      if (acc) return true;
      const can = await e.enforce(role, asset, action);
      if (can) return true;
    }, false);

    allowed ? next() : res.status(403).send("Forbidden").end();
  };
};
```

In this code snippet, we create a new Casbin enforcer using the newEnforcer function. Then, we call e.enforce(role, asset, action) on each user role, and return true as soon as the result of the e.enforce function is true. We return a 403 Forbidden response if the user is not allowed to perform the action on the asset, otherwise we call the next function to continue the middleware chain.
