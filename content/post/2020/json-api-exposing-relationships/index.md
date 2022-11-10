---
title: JSON:API - Exposing Relationships
stags: [JSON:API, REST]
author: "Yunier"
date: "2020-12-06"
description: "Guide on how to expose relationship link between resources"
series: ['JSON:API in .NET']
---

My [previous](/post/2020/json-api-exposing-the-customer-resource/index/) post on JSON:API exposed customers as an API resource, since then, I have updated the project to expose all remaining resources, that includes  Albums, Artists, Employees, Genres, Invoices, InvoiceItems, MediaTypes, Playlists, and Tracks. The time has come to expose the relationship that exist between these resource.

For this post, I will expose the one-to-many relationship that exist between artists and albums. To accomplish this task I will have to update the class ArtistServiceModelConfiguration by using the ToManyRelationship method exposed by JsonApiFramework in order to link one artist to many albums.

Here is current class definition for ArtistServiceModelConfiguration.

```c#
class ArtistServiceModelConfiguration : ResourceTypeBuilder<Artist>
{
    public ArtistServiceModelConfiguration()
    {
        // Ignore ER Core Navigation Properties
        this.Attribute(a => a.Albums)
            .Ignore();
    }
}
```

Here is the class definition afterwards.

```c#
class ArtistServiceModelConfiguration : ResourceTypeBuilder<Artist>
{
    public ArtistServiceModelConfiguration()
    {
        // Ignore ER Core Navigation Properties
        this.Attribute(a => a.Albums)
            .Ignore();

        this.ToManyRelationship<Album>(AlbumResourceKeyWords.ToManyRelationShipKey);
    }
}
```

With this change the API will expose the to-many part of the relationship between artist and album. Time to expose the to-one part of the relationship, for that I will update the class AlbumServiceModelConfiguration. 

Here is the class definition for AlbumServiceModelConfiguration.

```c#
class AlbumServiceModelConfiguration : ResourceTypeBuilder<Album>
{
    public AlbumServiceModelConfiguration()
    {
        // Ignore ER Core Navigation Properties
        this.Attribute(a => a.Artist)
            .Ignore();

        this.Attribute(a => a.Tracks)
            .Ignore();

        // Ignore Foreign Keys
        this.Attribute(a => a.ArtistId)
            .Ignore();
    }
}
```

Here is the class afterwards.

```c#
class AlbumServiceModelConfiguration : ResourceTypeBuilder<Album>
{
    public AlbumServiceModelConfiguration()
    {
        // Ignore ER Core Navigation Properties
        this.Attribute(a => a.Artist)
            .Ignore();

        this.Attribute(a => a.Tracks)
            .Ignore();

        // Ignore Foreign Keys
        this.Attribute(a => a.ArtistId)
            .Ignore();

        this.ToOneRelationship<Artist>(ArtistResourceKeyWords.ToOneRelationshipKey);
    }
}
```

Now the API will be able to expose the one-to-many relationship between artist and albums. With this changes JsonApiFramework will know how to build relationship links between artists and albums. Time to use the new configuration, I'll start by updating the GetResourceCollection and GetResource method that exist in the class AlbumResource and ArtistResource. The change will be be the same across all methods. I will update the fluent style API expose by JsonApiFramework by calling the AddRelationship method in the Relationships chain.

```c#
using var chinookDocumentContext = new ChinookJsonApiDocumentContext(currentRequestUri);
var document = chinookDocumentContext
    .NewDocument(currentRequestUri)
    .SetJsonApiVersion(JsonApiVersion.Version10)
        .Links()
            .AddSelfLink()
            .AddUpLink()
        .LinksEnd()
        .ResourceCollection(artistResourceCollection)
            .Relationships()
                .AddRelationship(AlbumResourceKeyWords.ToManyRelationShipKey, new[] { Keywords.Related })
            .RelationshipsEnd()
            .Links()
                .AddSelfLink()
            .LinksEnd()                          
        .ResourceCollectionEnd()
    .WriteDocument();
```

Now if I run the Web API project and go to any given artist, the API will expose a [related](https://tools.ietf.org/html/rfc4287#section-4.2.7.2) link to a collection of album.

```json
{
  "jsonapi": {
    "version": "1.0"
  },
  "links": {
    "self": "https://localhost:44323/artists/1",
    "up": "https://localhost:44323/artists"
  },
  "data": {
    "type": "artists",
    "id": "1",
    "attributes": {
      "name": "AC/DC"
    },
    "relationships": {
      "albums": {
        "links": {
          "related": "https://localhost:44323/artists/1/albums"
        }
      }
    },
    "links": {
      "self": "https://localhost:44323/artists/1"
    }
  }
}
```

and if I go to any given album, the API should expose a related link to a single artist.

```json
{
  "jsonapi": {
    "version": "1.0"
  },
  "links": {
    "self": "https://localhost:44323/albums/1",
    "up": "https://localhost:44323/albums"
  },
  "data": {
    "type": "albums",
    "id": "1",
    "attributes": {
      "title": "For Those About To Rock We Salute You"
    },
    "relationships": {
      "artist": {
        "links": {
          "related": "https://localhost:44323/albums/1/artist"
        }
      }
    },
    "links": {
      "self": "https://localhost:44323/albums/1"
    }
  }
}
```

Exposing the relationship alone is not enough, if you were to click any of those relationship links you would get a 404 error code, this is simply because we haven't updated the controllers to support these new links. Let's change that, time to update the Artist and Album controllers.

I'll start with the Artist controller. I'll update it by adding the following method.

```c#
[Route(ArtistRoutes.ArtistResourceToAlbumResourceCollection)]
public async Task<IActionResult> GetArtistResourceToAlbumResourceCollection(int resourceId)
{
    var document = await _artistResource.GetArtistResourceToAlbumResourceCollection(resourceId);
    return Ok(document);
}
```

GetArtistResourceToAlbumResourceCollection is a new method on the IArtistResource interface. Here is the updated interface definition.

```c#
public interface IArtistResource
{
    Task<Document> GetArtistResource(int resourceId);
    Task<Document> GetArtistResourceCollection();
    Task<Document> GetArtistResourceToAlbumResourceCollection(int resourceId);
}
```

Time to update the ArtistResource class by adding the concrete implementation of the GetArtistResourceToAlbumResourceCollection method.

```c#
public async Task<Document> GetArtistResourceToAlbumResourceCollection(int resourceId)
{
    var albumResourceCollection = await _mediator.Send(new GetArtistResourceToAlbumResourceCollectionCommand(resourceId));
    var currentRequestUri = _httpContextAccessor.HttpContext.GetCurrentRequestUri();

    using var chinookDocumentContext = new ChinookJsonApiDocumentContext(currentRequestUri);
    var document = chinookDocumentContext
        .NewDocument(currentRequestUri)
        .SetJsonApiVersion(JsonApiVersion.Version10)
            .Links()
                .AddSelfLink()
                .AddUpLink()
            .LinksEnd()
            .ResourceCollection(albumResourceCollection)
                .Links()
                    .AddSelfLink()
                .LinksEnd()
            .ResourceCollectionEnd()
        .WriteDocument();

    _logger.LogInformation("Request for {URL} generated JSON:API document {doc}", currentRequestUri, document);
    return document;
}
```

Perfect, the controller has been configured, now if I click on the related link to albums for artist with id 1 the API will respond with the following JSON:API document.

```json

{
  "jsonapi": {
    "version": "1.0"
  },
  "links": {
    "self": "https://localhost:44323/artists/1/albums",
    "up": "https://localhost:44323/artists/1"
  },
  "data": [
    {
      "type": "albums",
      "id": "1",
      "attributes": {
        "title": "For Those About To Rock We Salute You"
      },
      "links": {
        "self": "https://localhost:44323/albums/1"
      }
    },
    {
      "type": "albums",
      "id": "4",
      "attributes": {
        "title": "Let There Be Rock"
      },
      "links": {
        "self": "https://localhost:44323/albums/4"
      }
    }
  ]
}
```

Now I will need to make a similar change to the album controller.

```c#
// new controller method
[Route(AlbumRoutes.AlbumResourceToArtistResource)]
public async Task<IActionResult> GetAlbumResourceToArtistResource(int resourceId)
{
    var document = await _albumResource.GetAlbumResourceToArtistResource(resourceId);
    return Ok(document);
}

// updated IAlbumResource interface
public interface IAlbumResource
{
    Task<Document> GetAlbumResource(int resourceId);
    Task<Document> GetAlbumResourceCollection();
    Task<Document> GetAlbumResourceToArtistResource(int resourceId);
}

// concrete GetAlbumResourceToArtistResource implementation 
public async Task<Document> GetAlbumResourceToArtistResource(int resourceId)
{
    var artistResource = await _mediator.Send(new GetAlbumResourceToArtistResourceCommand(resourceId));
    var currentRequestUri = _httpContextAccessor.HttpContext.GetCurrentRequestUri();

    using var chinookDocumentContext = new ChinookJsonApiDocumentContext(currentRequestUri);
    var document = chinookDocumentContext
        .NewDocument(currentRequestUri)
        .SetJsonApiVersion(JsonApiVersion.Version10)
            .Links()
                .AddSelfLink()
                .AddUpLink()
            .LinksEnd()
            .Resource(artistResource)
                .Links()
                    .AddSelfLink()
                .LinksEnd()
            .ResourceEnd()
        .WriteDocument();

    _logger.LogInformation("Request for {URL} generated JSON:API document {doc}", currentRequestUri, document);
    return document;
}
```

Now if I click the related link to artist on an album resource I get the following JSON:API document as a response.

```json
{
  "jsonapi": {
    "version": "1.0"
  },
  "links": {
    "self": "https://localhost:44323/albums/4/artist",
    "up": "https://localhost:44323/albums/4"
  },
  "data": {
    "type": "artists",
    "id": "1",
    "attributes": {
      "name": "AC/DC"
    },
    "links": {
      "self": "https://localhost:44323/artists/1"
    }
  }
}
```

All that remains to do now is to expose all relationships. Something that I will accomplish before our next post.
