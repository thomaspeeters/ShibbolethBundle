ShibbolethBundle
================

This bundle adds a shibboleth authentication provider for your Symfony2 project.

Requirements
------------
* [PHP][@php] 5.3.3 and up.
* [Symfony 2.1][@symfony]

Installation
------------

ShibbolethBundle is composer-friendly.

### 1. Add ShibbolethBundle in your composer.json

```js
    "require": {
        ...
        "kuleuven/shibboleth-bundle": "dev-master"
        ...
    },
   "repositories": [
        {
            "type": "vcs",
            "url": "git@github.com:rmoreas/ShibbolethBundle.git"
        }
    ],	
```
Now tell composer to download the bundle by running the command:

```bash
    php composer.phar update kuleuven/shibboleth-bundle
```

Composer will install the bundle to your project's vendor/kuleuven directory..

### 2. Enable the bundle

Instantiate the bundle in your kernel:

```php
// app/AppKernel.php
<?php
    // ...
    public function registerBundles()
    {
        $bundles = array(
            // ...
            new KULeuven\ShibbolethBundle\ShibbolethBundle(),
        );
    }
```

Configuration
-------------

### 1. Enable lazy shibboleth autentication in Apache

Add following lines to the .htaccess file in your projects web folder

```apache
    # web/.htaccess
	AuthType shibboleth
	ShibRequireSession Off
	ShibUseHeaders On
	require shibboleth
```

### 2. Setup authentication firewall 

```yml
	# app/config/security.yml
	security:
		firewalls:
			secured_area:
				pattern:    ^/secured
				shibboleth: ~
                logout:
                    path: /secured/logout
                    target: /
                    success_handler: security.logout.handler.shibboleth
```

### 3. Shibboleth configuration

Possible configuration parameters are:

```yml
	# app/config/config.yml
	shibboleth:
		handler_path: /Shibboleth.sso
		secured_handler: true
		session_initiator_path: /Login			
```

The above listed configuration values are the default values. To use the defaults, simply use the following line in your config:

```yml
	# app/config/config.yml
	shibboleth: ~
```

User Provider
-------------

This bundle doesn't include any User Provider, but you can implement your own.

If you store users in a database, they can be created on the fly when a users logs on for the first time on your application. Your UserProvider needs to implement the `KULeuven\ShibbolethBundle\Security\ShibbolethUserProviderInterface` interface.

### Example

This example uses Propel ORM to store users.

```php
	<?php 
	namespace YourProjectNamespace\Security;

	use YourProjectNamespace\Model\User;
	use YourProjectNamespace\Model\UserQuery;

	use KULeuven\ShibbolethBundle\Security\ShibbolethUserProviderInterface;
	use KULeuven\ShibbolethBundle\Security\ShibbolethUserToken;

	use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
	use Symfony\Component\Security\Core\User\UserProviderInterface;
	use Symfony\Component\Security\Core\User\UserInterface;
	use Symfony\Component\Security\Core\Exception\UsernameNotFoundException;
	use Symfony\Component\Security\Core\Exception\UnsupportedUserException;

	class UserProvider implements ShibbolethUserProviderInterface
	{
		public function loadUserByUsername($username)
		{
			$user = UserQuery::create()->findOneByUsername($username);
			if($user){
				return $user;
			} else{
				throw new UsernameNotFoundException("User ".$username. " not found.");
			}
		}
		
		public function createUser(ShibbolethUserToken $token){
			// Create user object using shibboleth attributes stored in the token. 
			// 
			$user = new User();
			$user->setUid($token->getUsername());
			$user->setSurname($token->getSurname());
			$user->setGivenName($token->getGivenName());
			$user->setMail($token->getMail());
			// If you like, you can also add default roles to the user based on shibboleth attributes. E.g.:
			if ($token->isStudent()) $user->addRole('ROLE_STUDENT');
			elseif ($token->isStaff()) $user->addRole('ROLE_STAFF');
			else $user->addRole('ROLE_GUEST');
			
			$user->save();
			return $user;
		}

		public function refreshUser(UserInterface $user)
		{
			if (!$user instanceof User) {
				throw new UnsupportedUserException(sprintf('Instances of "%s" are not supported.', get_class($user)));
			}

			return $this->loadUserByUsername($user->getUsername());
		}

		public function supportsClass($class)
		{
			return $class === 'YourProjectNamespace\Model\User';
		}
	}
```

