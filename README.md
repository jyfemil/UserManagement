# Sandstorm.UserManagement Neos / Flow Package

## Features
This package works in Neos CMS and Flow and provides the following functionalities:

* Registration of (frontend) users via a registration form
* Sending out an e-mail for account confirmation
* Login of registered (frontend) users via a login form
* "Forgotten password" with password reset e-mail

# Configuration

## Setup
Run ./flow doctrine:migrate after you add this package to install its model. The package automatically exposes its routes
via auto-inclusion in the package settings.
Attention: Any routes defined in the global `Routes.yaml` are loaded before this package's routes, so they may be overriden.
This is especially true for the default Flow subroutes, so make sure you have removed those from your global `Routes.yaml`.
If you can't remove them, just include the subroutes for this package manually before the Flow subroutes.

## Basic configuration options
These are the basic configuration options for e-mails, timeouts etc. You will usually want to adapt these to your application.
```
Sandstorm:
  UserManagement:
    # Validity timespan for the activation token for newly registered users.
    activationTokenTimeout: '2 days'
    # Validity timespan for the token used to reset passwords.
    resetPasswordTokenTimeout: '4 hours'
    # Email settings
    email:
      # Sender Address
      senderAddress: 'test@example.com'
      # Sender name - will be merged with senderAddress to something
      # like "Sandstorm Usermanagement Package <test@example.com>"
      senderName: 'Sandstorm Usermanagement Package'
      # Subject line for the account confirmation email
      subjectActivation: 'Please confirm your account'
      # Subject line for the password reset email
      subjectResetPassword: 'Password reset'
      # Template package to read the email templates from - this can be used to override
      # e-mail templates. They are expected in the package given here, in the folder
      # <Package>/Resources/Private/EmailTemplates.
      templatePackage: 'Sandstorm.UserManagement'
    # An array of roles which are assigned to users after they activate their account.
    rolesForNewUsers: []
```

## Configuring SwiftMailer
The UserManagement package requires SwiftMailer to send out e-mails. Please check the swiftmailer package's
configuration options (https://github.com/neos/swiftmailer) in order to configure SMTP credentials.

## Additional Settings for usage in Neos
Add the following to your package's (or the global) `Settings.yaml`. This creates a separate authentication provider so Neos can
distinguish between frontend and backend logins.
TODO: Check if the default Flow provider needs to be disabled.

```

TYPO3:
  Flow:
    security:
      authentication:
        providers:
          'Typo3BackendProvider':
            requestPatterns:
              'Sandstorm\UserManagement\Security\NeosRequestPattern': 'backend'
          'Sandstorm.UserManagement:Login':
            provider: 'PersistedUsernamePasswordProvider'
            requestPatterns:
              'Sandstorm\UserManagement\Security\NeosRequestPattern': 'frontend'

```

Also, you should switch the implementation of the Redirect and User Creation Services. Add this to your `Objects.yaml`:
```
# Use the Neos services
Sandstorm\UserManagement\Domain\Service\RedirectTargetServiceInterface:
  className: 'Sandstorm\UserManagement\Domain\Service\Neos\NeosRedirectTargetService'
Sandstorm\UserManagement\Domain\Service\UserCreationServiceInterface:
  className: 'Sandstorm\UserManagement\Domain\Service\Neos\NeosUserCreationService'
```

# Usage

## Creating users via the CLI
The package exposes a command to create users. You can run

`./flow sandstormuser:create test@example.com password firstName lastName`

to create a test user. This will create a Neos user if you're using the package in Neos. You can assign
roles to the new user in the Neos backend afterwards. It doesn't work yet for standalone usage in Flow (see TODOS).

## Redirect after login/logout
### Via configuration
To define where users should be redirected after they log in or out, you can set some config options:
```
Sandstorm:
  UserManagement:
    redirect:
    # To activate redirection, make these settings:
      afterLogin:
        action: 'action'
        controller: 'Controller'
        package: 'Your.Package'
      afterLogout:
        action: 'action'
        controller: 'Controller'
        package: 'Your.Package'
```

### Via node properties
When using the package within Neos, you have another possibility: you can set properties on the LoginForm node type.
The pages you link here will be shown after users log in or out. Please note that when a login/logout form is displayed
on a restricted page: in that case you MUST set a redirect target, otherwise you will receive an error message on logout.
If the redirection is configured via Settings.yaml, they will take precedence over the configuration at the node.
You can, of course, set these properties from TypoScript also if you have a login/logout form directly in you template:
```
loginform = Sandstorm.UserManagement:LoginForm {
  // This should be set, or there will be problems when you have multiple plugins on a page
  argumentNamespace = 'login'
  // Redirect to the parent page automatically after logout
  redirectAfterLogout = ${q(documentNode).parent().get(0)}
}
```

### Via custom RedirectTargetService
If redirecting to a specific controller method is still not enough for you, you can simply roll your own implementation of the
`RedirectTargetServiceInterface`. Just add the implementation within your own package and add the following lines to your `Objects.yaml`.
Mind the package loading order, you package should require sandstorm/usermanagement in its composer.json.
```
Sandstorm\UserManagement\Domain\Service\RedirectTargetServiceInterface:
  className: 'Your\Package\Domain\Service\YourCustomRedirectTargetService'
```

## Checking for a logged-in user in your templates
There is a ViewHelper available that allows you to check if somebody is logged into the frontend. Here's an example:

```
{namespace usermanagement=Sandstorm\UserManagement\ViewHelpers}

<usermanagement:ifAuthenticated authenticationProviderName="Sandstorm.UserManagement:Login">
  <f:then>
    You are currently logged in.
  </f:then>
  <f:else>
    You are not logged in!
  </f:else>
</usermanagement:ifAuthenticated>

```

# Extending the package

## Changing / overriding templates
You can change any template via the default method using `Views.yaml`. Please see
http://flowframework.readthedocs.io/en/stable/TheDefinitiveGuide/PartIII/ModelViewController.html#configuring-views-through-views-yaml.
Here's an example how to plug your own login template:
```
-
  requestFilter: 'isPackage("Sandstorm.UserManagement") && isController("Login") && isAction("login")'
  options:
    templatePathAndFilename: 'resource://Your.Package/Private/Templates/Login/Login.html'
    partialRootPaths: ['resource://Your.Package/Private/Partials']
    layoutRootPaths: ['resource://Your.Package/Private/Layouts']
```

## Overriding e-mail templates
As documented in the configuration options above, overriding e-mail templates is easy:
* Copy the `EmailTemplates` folder from the UserManagement's `Resources/Private` folder into your
  own package and modify the templates to your heart's desire.
* Then, set the `email.templatePackage` configuration option to that package's name. Done!

## Changing or adding properties to the registration flow; changing the User model
You might want to add additional information to the user model. This can be done by extending
the User model delivered with this package and adding properties as you like. You will then
need to switch out the implementation of `UserCreationServiceInterface` to get control over
the creation process. This can be done via `Objects.yaml`:
```
Sandstorm\UserManagement\Domain\Service\UserCreationServiceInterface:
  className: 'Your\Package\Domain\Service\YourCustomUserCreationService'
```

TODO: Document registrationFlow.attributes

# Known issues

Feel free to submit issues/PRs :)

# TODOs

* We haven't described all features in detail yet.
* An important missing feature: configuring password restrictions (8 chars min, a smiley and a celtic rune, ...)
* I18N for Templates.
* Tests.

# FAQ

* *What happens if the user did not receive the registration email?*
  Just tell the user to register again. In this case, previous unfinished registrations are discarded.

# License
MIT.
https://opensource.org/licenses/MIT
