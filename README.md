# Symfony Base64 MediaObject VichUploader

This bundle integrates VichUploaderBundle with Symfony, enabling file uploads using Base64 encoding.

# Install the bundle with the help of Composer:

```docker compose exec php composer require vich/uploader-bundle```

During the installation, you'll be prompted to execute the recipe:

    Do you want to execute this recipe?
    [y] Yes
    [n] No
    [a] Yes for all packages, only for the current installation session
    [p] Yes permanently, never ask again for this project
    (defaults to n): y     

Type ```y``` and press Enter to proceed.

# Configuration

Configure the bundle in config/packages/vich_uploader.yaml:

    vich_uploader:
        db_driver: orm
        metadata:
            type: attribute
        mappings:
            media_object:
               uri_prefix: /media
               upload_destination: '%kernel.project_dir%/public/media'
               # Will rename uploaded files using a uniqueid as a prefix.
               namer: Vich\UploaderBundle\Naming\OrignameNamer  

# Serializer

Create a custom serializer for MediaObject in api/src/Serializer/MediaObjectNormalizer.php:

```php
<?php

declare(strict_types=1);

namespace App\Serializer;

use App\Entity\MediaObject;
use Symfony\Component\Serializer\Exception\ExceptionInterface;
use Symfony\Component\Serializer\Normalizer\ContextAwareNormalizerInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareInterface;
use Symfony\Component\Serializer\Normalizer\NormalizerAwareTrait;
use Vich\UploaderBundle\Storage\StorageInterface;

final class MediaObjectNormalizer implements ContextAwareNormalizerInterface, NormalizerAwareInterface
{
    use NormalizerAwareTrait;

    private const ALREADY_CALLED = 'MEDIA_OBJECT_NORMALIZER_ALREADY_CALLED';

    public function __construct(private readonly StorageInterface $storage)
    {
    }

    /**
     * @throws ExceptionInterface
     */
    public function normalize($object, ?string $format = null, array $context = []): array|string|int|float|bool|\ArrayObject|null
    {
        $context[self::ALREADY_CALLED] = true;

        $object->contentUrl = $this->storage->resolveUri($object, 'file');

        return $this->normalizer->normalize($object, $format, $context);
    }

    public function supportsNormalization($data, ?string $format = null, array $context = []): bool
    {
        if (isset($context[self::ALREADY_CALLED])) {
            return false;
        }

        return $data instanceof MediaObject;
    }
}
```


# Utils

# Utility classes for handling Base64 files:

# Base64FileExtractor

Extracts the Base64 string from a Base64 encoded content.

// api/src/Utils/Base64FileExtractor.php

```php
<?php
declare(strict_types=1);

namespace App\Utils;

class Base64FileExtractor
{
    public function extractBase64String(string $base64Content): string
    {
        $data = explode( ';base64,', $base64Content);
        return $data[1];
    }

}
```
# ExtensionBase64
Extracts the file extension from a Base64 encoded string.

// api/src/Utils/ExtensionBase64.php
````php
<?php
declare(strict_types=1);

namespace App\Utils;

abstract class ExtensionBase64
{
    public function extensionBase64(string $base64): string
    {
        preg_match("/\/(.*?);/", $base64, $match);

        return ".$match[1]";
    }
}
````
# ProcessFile
Processes the Base64 file and sets it on the MediaObject.

// api/src/Utils/ProcessFile.php

````php
<?php
declare(strict_types=1);

namespace App\Utils;

use App\Entity\MediaObject;

class ProcessFile extends ExtensionBase64
{
    public function processFile(MediaObject $mediaObject): void
    {
        $base64 = $mediaObject->getImage();

        $base64Image = (new Base64FileExtractor)->extractBase64String($base64);
        $imageFile = new UploadedBase64File($base64Image, $this->extensionBase64($base64));
        $mediaObject->setFile($imageFile);
    }
}
````
# UploadedBase64File

Represents a file uploaded via Base64 encoding.

// api/src/Utils/UploadedBase64File.php
````php
<?php

declare(strict_types=1);

namespace App\Utils;

use Symfony\Component\HttpFoundation\File\UploadedFile;

class UploadedBase64File extends UploadedFile
{
    public function __construct(string $base64String, string $originalName)
    {
        $filePath = tempnam(sys_get_temp_dir(), 'UploadedFile');
        $data = base64_decode($base64String);
        file_put_contents($filePath, $data);
        $error = null;
        $mimeType = null;

        parent::__construct($filePath, $originalName, $mimeType, $error, true);
    }
}
````

# Controller

Create a controller for handling Base64 media uploads in 

api/src/Controller/Media/CreateMediaBase64Action.php
````php
<?php

declare(strict_types=1);

namespace App\Controller\Media;

use App\Controller\Base\AbstractController;
use App\Entity\MediaObject;
use App\Utils\ProcessFile;

class CreateMediaBase64Action extends AbstractController
{
    public function __invoke(MediaObject $mediaObject, ProcessFile $processFile): MediaObject
    {
        $this->validate($mediaObject);
        $processFile->processFile($mediaObject);

        return $mediaObject;
    }
}
````

# Entity

Define the MediaObject entity in api/src/Entity/MediaObject.php:
````php
<?php

namespace App\Entity;

use ApiPlatform\Metadata\ApiProperty;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;
use App\Controller\Media\CreateMediaBase64Action;
use App\Repository\MediaObjectRepository;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\HttpFoundation\File\File;
use Symfony\Component\Serializer\Annotation\Groups;
use Symfony\Component\Validator\Constraints as Assert;
use Vich\UploaderBundle\Mapping\Annotation as Vich;

#[Vich\Uploadable]
#[ORM\Entity(repositoryClass: MediaObjectRepository::class)]
#[ApiResource(
    operations: [
        new Get(),
        new GetCollection(),
        new Post(
            controller: CreateMediaBase64Action::class
        )
    ],
    normalizationContext: ['groups' => ['media_object:read']],
    denormalizationContext: ['groups' => ['media_object:write']]
)]
class MediaObject
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    #[ApiProperty(types: ['https://schema.org/contentUrl'])]
    #[Groups(['media_object:read'])]
    public ?string $contentUrl = null;

    #[Groups(['media_object:write'])]
    private ?string $image = null;

    #[Vich\UploadableField(mapping: "media_object", fileNameProperty: "filePath")]
    #[Assert\NotNull(groups: ['media_object_create'])]
    #[Groups(['media_object:read'])]
    private ?File $file = null;

    #[ORM\Column(nullable: true)]
    public ?string $filePath = null;

    public function getFile(): ?File
    {
        return $this->file;
    }

    public function setFile(?File $file): self
    {
        $this->file = $file;

        return $this;
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getImage(): ?string
    {
        return $this->image;
    }

    public function setImage(?string $image): self
    {
        $this->image = $image;

        return $this;
    }

}
````
This documentation provides a step-by-step guide to setting up the SymfonyBase64VichUploader bundle, including installation, configuration, custom serialization, utility classes, and the MediaObject entity.

               
    
