# Decouplate domain user from Symfony

Usually, in Symfony, users are handled by a User class implementing [`UserInterface`](https://github.com/symfony/symfony/blob/master/src/Symfony/Component/Security/Core/User/UserInterface.php) and contain Symfony related stuff like `eraseCredential` or `getRoles` methods. If you are using something like DDD or Hexagonal Architecture, you may want your user class don't be related to Symfony.

## Adapter

Your domain user model (entity):

```php
<?php
class Client()
{
    private $id;
    private $email;
    private $password;
}
```

Your Symfony user adapter:

```php
<?php
class SymfonyClient implement UserInterface
{
    private $client;

    public function __construct(Client $model)
    {
        $this->model = $client;
    }

    public function getRoles()
    {
        return ['ROLE_USER'];
    }
    
    public function getPassword()
    {
        return $this->model->getPassword();
    }
    
    public function getSalt()
    {
        return null;
    }
    
    public function getUsername()
    {
        return $this->model->getEmail();
    }
    
    public function eraseCredentials()
    {
    }
}
```

Then create your own user provider to return your adapter instead of your entity:

```php
<?php
class SymfonyClientProvider extends UserProviderInterface
{
    private $repository;

    public function ___construct($repository)
    {
        $this->repository = $repository;
    }

    public function loadUserByUsername($username)
    {
        $client = $this->repository->findOneByUsername($username);
        
        if (null === $client) { throw new UsernameNotFoundException('User not found.'); }
        
        return new SymfonyClient($client);
    }
    
    public function refreshUser(UserInterface $user)
    {
        if (!$user instanceof SymfonyClient) { throw new UnsupportedUserException('User not supported.'); }
        
        return $this->loadUserByUsername($user->getUsername());
    }
    
    public function supportsClass($class)
    {
        return SymfonyClient::class === $class;
    }
}
```

declare the service:

```yaml
services:
    app.symfony_client_provider:
        class: App\SymfonyClientProvider
        arguments:
            - '@app.client_repository'
```

and configure your provider in `security.yml`:

```yaml
security:
    providers:
        client:
            id: app.symfony_client_provider
    firewalls:
        main:
            form_login:
                provider: client
```
            id: app.symfony_client_provider
