---
title: JSON&#58;API in .NET Core - Exposing Relationships
layout: post
categories: [.NET Core, JSON&#58;API, JsonApiFramework, APIs, REST]
image: /assets/img/json-api/chinook-database-entities.PNG
description: "Part four of my blog series on building a .NET Core Web Api using JSON&#58;API"
---

My [last](https://www.yunier.dev/2020/10/31/JSON-API-Adding-Customer-Resource.html) post on JSON:API exposed customer as an API resource, since then, I have updated the project to expose all resources, that includes, Album, Artist, Employee, Genre, Invoice, InvoiceItems, MediaTypes, Playlist, and Tracks. The time has come to expose relationships between these resource. 

For this post, I will expose a one-to-many relationship between artist and albums. To accomplish I will need to update ArtistServiceModelConfiguration by using the ToManyRelationship method to link one artist to many albums. 

Here is current class definition.

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

And here is the class definition afterwards, as you can see, we now have a the to-many part of the relationship between the Artist resource and Album resource.

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

Time to add the to-one part of the relationship, for that, I will need to update the class AlbumServiceModelConfiguration. Here is the current class definition.

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

And here is the class afterwards.

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

I've successfully created a one-to-many relationship between artist and albums. Now JsonApiFramework will know how to build links between the Artist and Album resources. Time to use that configuration. I'll open the ArtistResource class to update the methods GetArtistResourceCollection and GetArtistResource by using the Relationships method that is available under the JsonApiFramework fluent style API.

```c#
    // GetArtistResourceCollection, a similar update will be done to GetArtistResource
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

If I run the project, I should see "albums" exposed as relationship under each artist resource and "artist" exposed as a relationship under each album.

![artist-to-album-relationship](/assets/img/json-api/artist-to-album-relationship.PNG)

![artist-to-album-relationship](/assets/img/json-api/album-to-artist-relationship.PNG)