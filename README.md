# 🏛️ Symfony 6 + API Platform — Clean Architecture & DDD

Une refonte d'application legacy comme Hermes 2 ne se fait pas en ajoutant
des features sur du code existant — elle se fait en posant une architecture
nouvelle, modulaire, testable, que l'on peut faire évoluer sans tout casser.
Ce projet implémente cette architecture en Symfony 6 + API Platform 3,
avec une structuration DDD par contextes métier.

---

## Architecture par contextes — structure DDD

```
  src/
  │
  ├── Common/                    ← éléments partagés entre contextes
  │   ├── Application/
  │   │   └── Dto/
  │   │       └── RequestDtoInterface.php
  │   ├── Domain/
  │   │   └── Bus/               ← interfaces Command/Query bus
  │   └── Infrastructure/
  │       └── Symfony/           ← résolveurs Symfony custom
  │
  ├── User/                      ← contexte User (DDD Bounded Context)
  │   ├── Application/
  │   │   ├── Command/           ← CreateUserCommand, UpdateUserCommand
  │   │   ├── Query/             ← GetUserQuery, ListUsersQuery
  │   │   └── Handler/           ← un handler par command/query
  │   ├── Domain/
  │   │   ├── Entity/
  │   │   │   └── User.php       ← entité domain pure, sans annotation Doctrine
  │   │   ├── Repository/
  │   │   │   └── UserRepositoryInterface.php  ← contrat, pas d'EF
  │   │   └── Event/
  │   │       └── UserCreatedEvent.php
  │   └── Infrastructure/
  │       ├── Persistence/
  │       │   └── DoctrineUserRepository.php   ← implémente l'interface
  │       └── Api/
  │           └── UserController.php           ← entrée API Platform
  │
  ├── Messaging/                 ← contexte Messagerie
  └── Authentication/            ← contexte Auth (JWT)
```

---

## Entité Domain — pure, sans annotation Doctrine

```php
// User/Domain/Entity/User.php
// Zéro dépendance à Doctrine ou Symfony dans le Domain

declare(strict_types=1);

namespace App\User\Domain\Entity;

use App\User\Domain\Event\UserCreatedEvent;

final class User
{
    private array $domainEvents = [];

    public function __construct(
        private readonly UserId    $id,
        private string             $email,
        private string             $passwordHash,
        private UserRole           $role = UserRole::USER,
        private \DateTimeImmutable $createdAt = new \DateTimeImmutable(),
    ) {}

    public static function create(UserId $id, string $email, string $passwordHash): self
    {
        $user = new self($id, $email, $passwordHash);
        $user->raise(new UserCreatedEvent($id, $email));
        return $user;
    }

    public function changeEmail(string $newEmail): void
    {
        if (!filter_var($newEmail, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Email invalide : $newEmail");
        }
        $this->email = $newEmail;
    }

    public function promote(UserRole $role): void
    {
        $this->role = $role;
    }

    // Domain Events — collectés, dispatchés par l'Infrastructure
    public function raise(object $event): void
    {
        $this->domainEvents[] = $event;
    }

    public function pullDomainEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }

    // Getters
    public function getId(): UserId    { return $this->id; }
    public function getEmail(): string { return $this->email; }
    public function getRole(): UserRole { return $this->role; }
}
```

---

## Repository — interface dans le Domain, implémentation dans l'Infrastructure

```php
// User/Domain/Repository/UserRepositoryInterface.php
interface UserRepositoryInterface
{
    public function findById(UserId $id): ?User;
    public function findByEmail(string $email): ?User;
    public function save(User $user): void;
    public function delete(UserId $id): void;
}

// User/Infrastructure/Persistence/DoctrineUserRepository.php
final class DoctrineUserRepository implements UserRepositoryInterface
{
    public function __construct(
        private readonly EntityManagerInterface $em
    ) {}

    public function findById(UserId $id): ?User
    {
        return $this->em->find(UserEntity::class, $id->value());
    }

    public function save(User $user): void
    {
        // Mapper Domain → Entity Doctrine
        $entity = UserEntityMapper::fromDomain($user);
        $this->em->persist($entity);
        $this->em->flush();

        // Dispatcher les Domain Events après la persistance
        foreach ($user->pullDomainEvents() as $event) {
            $this->eventDispatcher->dispatch($event);
        }
    }
}
```

---

## Command Handler — orchestration sans couplage à l'Infrastructure

```php
// User/Application/Handler/CreateUserHandler.php
final readonly class CreateUserHandler
{
    public function __construct(
        private UserRepositoryInterface $users,
        private PasswordHasherInterface $hasher,
    ) {}

    public function __invoke(CreateUserCommand $command): UserId
    {
        // Vérifier unicité
        if ($this->users->findByEmail($command->email) !== null) {
            throw new UserAlreadyExistsException($command->email);
        }

        // Créer l'entité domain
        $id   = UserId::generate();
        $hash = $this->hasher->hash($command->password);
        $user = User::create($id, $command->email, $hash);

        // Persister (Domain Events dispatchés dans le repository)
        $this->users->save($user);

        return $id;
    }
}
```

---

## Tests PHPUnit — handler isolé sans base de données

```php
class CreateUserHandlerTest extends TestCase
{
    private MockObject $repoMock;
    private MockObject $hasherMock;
    private CreateUserHandler $handler;

    protected function setUp(): void
    {
        $this->repoMock   = $this->createMock(UserRepositoryInterface::class);
        $this->hasherMock = $this->createMock(PasswordHasherInterface::class);
        $this->handler    = new CreateUserHandler($this->repoMock, $this->hasherMock);
    }

    public function testCreateUserSuccessfully(): void
    {
        $this->repoMock->method('findByEmail')->willReturn(null);
        $this->hasherMock->method('hash')->willReturn('hashed_password');
        $this->repoMock->expects($this->once())->method('save');

        $id = ($this->handler)(new CreateUserCommand('user@test.com', 'secret123'));

        $this->assertInstanceOf(UserId::class, $id);
    }

    public function testThrowsIfEmailAlreadyExists(): void
    {
        $existingUser = User::create(UserId::generate(), 'user@test.com', 'hash');
        $this->repoMock->method('findByEmail')->willReturn($existingUser);

        $this->expectException(UserAlreadyExistsException::class);
        ($this->handler)(new CreateUserCommand('user@test.com', 'secret'));
    }
}
```

---

## Ce que j'ai appris

Séparer l'entité Domain de l'entité Doctrine semble être du travail en double.
En pratique, c'est ce qui permet de faire évoluer le schéma de base de données
sans toucher aux règles métier — et inversement. Sur une refonte comme Hermes 2,
c'est crucial : on peut migrer progressivement le schéma SQL sans régression
sur la logique applicative, car elles sont dans des couches indépendantes.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
