Atelier : Mise en pratique des relations "One-to-Many" et "Many-to-Many" avec Symfony

## Objectifs :
### Objectifs de l'atelier
- Utiliser la commande make:entity de Symfony pour créer des relations entre entités.
- Implémenter une relation One-to-Many entre Utilisateur et Conge.
- Implémenter une relation Many-to-Many entre Utilisateur et Groupe.
- Tester les relations avec des fixtures et des manipulations dans des contrôleurs.

## Explication :
### Noms des classes :
- **Utilisateur** : représente la classe User (nom adapté en français).
- **Conge** : représente les congés associés à un utilisateur.
- **Groupe** : représente un groupe auquel les utilisateurs peuvent appartenir.

### Relations :
- Utilisateur "1" -- "0..*" Conge : Un utilisateur peut avoir plusieurs congés, mais chaque congé est associé à un seul utilisateur.
- Utilisateur "0..*" -- "0..*" Groupe : Un utilisateur peut appartenir à plusieurs groupes, et chaque groupe peut inclure plusieurs utilisateurs.

---

## Étape 1 : Préparation
### Avant de commencer :
- Assurez-vous que le projet Symfony est configuré.
- La base de données doit être connectée et prête à recevoir des entités. Vérifiez cela dans le fichier `.env`.
- L'entité Utilisateur et l'entité Conge sont déjà en place.
- Nous allons créer l'entité Groupe et mettre en place la relation Many-to-Many avec l'entité Utilisateur.

---

## Étape 2 : Créer l'entité Groupe et établir la relation Many-to-Many

### 1. Créer l'entité Groupe
Utilisez la commande suivante pour créer l'entité Groupe :
```bash
php bin/console make:entity Groupe

Ajoutez les champs suivants lors du prompt interactif :
nom : string (nom du groupe, ex. "Administrateurs", "Employés").
Relation avec Utilisateur :
Type : ManyToMany
Propriétaire : Utilisateur (relation dans Utilisateur).
Symfony mettra à jour les fichiers de migration et la classe Utilisateur.

2. Mettre à jour l'entité Utilisateur pour la relation Many-to-Many
Ajoutez la relation ManyToMany dans l'entité Utilisateur (si elle n'est pas déjà présente).
Voici à quoi pourrait ressembler la mise à jour de l'entité Utilisateur :
php
Copier le code
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use App\Repository\UtilisateurRepository;

#[ORM\Entity(repositoryClass: UtilisateurRepository::class)]
#[ORM\Table(name: '`utilisateur`')]
class Utilisateur
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 255)]
    private ?string $nom = null;

    #[ORM\Column(length: 255)]
    private ?string $prenom = null;

    #[ORM\Column(length: 255)]
    private ?string $email = null;

    #[ORM\Column(length: 255)]
    private ?string $motDePasse = null;

    #[ORM\ManyToMany(targetEntity: Groupe::class)]
    #[ORM\JoinTable(name: 'utilisateur_groupe')]
    private Collection $groupes;

    public function __construct()
    {
        $this->groupes = new ArrayCollection();
    }

    // Getter et setter pour 'groupes'
    public function getGroupes(): Collection
    {
        return $this->groupes;
    }

    public function addGroupe(Groupe $groupe): self
    {
        if (!$this->groupes->contains($groupe)) {
            $this->groupes[] = $groupe;
        }

        return $this;
    }

    public function removeGroupe(Groupe $groupe): self
    {
        $this->groupes->removeElement($groupe);

        return $this;
    }

    // ... autres getters et setters ...
}


Étape 3 : Appliquer les migrations
Générez les fichiers de migration pour refléter les nouvelles relations dans la base de données :
bash
Copier le code
php bin/console make:migration


Exécutez les migrations pour mettre à jour la base de données :
bash
Copier le code
php bin/console doctrine:migrations:migrate



Étape 4 : Mise à jour des fixtures pour ajouter des groupes à l’utilisateur
Ensuite, dans votre classe AppFixtures, vous devez générer des groupes et les associer à des utilisateurs.
Voici une mise à jour de votre classe AppFixtures pour inclure des groupes dans les données fictives :
namespace App\DataFixtures;

use App\Entity\Groupe;
use App\Factory\CongeFactory;
use App\Factory\UserFactory;
use Doctrine\Bundle\FixturesBundle\Fixture;
use Doctrine\Persistence\ObjectManager;
use Faker\Factory;

class AppFixtures extends Fixture
{
    public function load(ObjectManager $manager): void
    {
        // Crée des utilisateurs fictifs
        $users = UserFactory::new()->createMany(10, function () {
            $faker = Factory::create();
            return [
                'nom' => $faker->lastName('fr_FR'),
                'prenom' => $faker->firstName('fr_FR'),
                'email' => $faker->unique()->safeEmail,
                'password' => password_hash('password', PASSWORD_BCRYPT),
            ];
        });

        // Crée des groupes fictifs
        $groupes = [
            'Administrateurs',
            'Employés',
            'Managers'
        ];

        $groupeEntities = [];
        foreach ($groupes as $groupeNom) {
            $groupe = new Groupe();
            $groupe->setNom($groupeNom);
            $manager->persist($groupe);
            $groupeEntities[] = $groupe; // Stocke les groupes créés
        }

        // Associe chaque utilisateur à des groupes fictifs
        foreach ($users as $user) {
            // Ajout aléatoire de groupes à chaque utilisateur
            $randomGroups = array_rand($groupeEntities, rand(1, count($groupeEntities)));
            if (!is_array($randomGroups)) {
                $randomGroups = [$randomGroups];
            }

            foreach ($randomGroups as $groupIndex) {
                $user->addGroupe($groupeEntities[$groupIndex]);
            }

            // Crée des congés fictifs pour chaque utilisateur
            $typesDeConges = [
                'Congé annuel',
                'Congé maladie',
                'Congé sans solde',
                'Congé maternité/paternité',
                'RTT',
                'Congé sabbatique'
            ];

            $totalDays = 0; // Initialise le total des jours de congé pour l'utilisateur
            foreach ($typesDeConges as $typeDeConge) {
                $days = rand(1, 30); // Génère un nombre aléatoire de jours de congé entre 1 et 30
                if ($totalDays + $days > 30) {
                    break; // Arrête d'ajouter des congés si le total dépasse 30 jours
                }
                $dateDebut = new \DateTime(sprintf('-%d days', rand(1, 30))); // Génère une date de début aléatoire
                CongeFactory::new()->create([
                    'type' => $typeDeConge, // Type de congé
                    'dateDebut' => $dateDebut, // Date de début du congé
                    'dateFin' => (clone $dateDebut)->modify(sprintf('+%d days', $days)), // Date de fin du congé
                    'statut' => rand(0, 1) ? 'approuvé' : 'en attente', // Statut aléatoire du congé
                    'user' => $user // Associe le congé à l'utilisateur
                ]);
                $totalDays += $days; // Ajoute les jours de congé au total
            }
        }

        $manager->flush();
    }
}




Étape 5 : Tester les relations dans un contrôleur
Créez un contrôleur pour vérifier les relations en récupérant les utilisateurs avec leurs congés et groupes.
php
Copier le code
namespace App\Controller;

use App\Repository\UtilisateurRepository;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;

class UtilisateurController extends AbstractController
{
    #[Route('/utilisateurs', name: 'utilisateurs_liste')]
    public function index(UtilisateurRepository $repository): Response
    {
        $utilisateurs = $repository->findAll();

        return $this->render('utilisateur/index.html.twig', [
            'utilisateurs' => $utilisateurs,
        ]);
    }
}


Étape 6 : Vérification finale
Lancez le serveur Symfony :
bash
Copier le code
symfony server:start
Accédez à l'URL /utilisateurs pour vérifier que les utilisateurs, leurs congés et leurs groupes s’affichent correctement.

Résultat attendu
Vous aurez une application fonctionnelle avec :
Une relation One-to-Many entre Utilisateur et Conge (déjà en place).
Une relation Many-to-Many entre Utilisateur et Groupe.
Des données fictives pour valider vos relations.



