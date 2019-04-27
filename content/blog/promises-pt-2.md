---
title: "Promises Pt 2"
date: 2019-04-26T23:30:48-04:00
draft: true
---

> <h1 style="color: #aaa;">We have overcome</h1>

If you remember my last post, I was having trouble with Promises. I didn't know it at the time, but I was struggling with structuring my code to fully leverage the chaining power of Promises.

So with my new findings I'll explain the same GET request, now fully powered by promises!

## Environment

We are running a pretty bare bones Express.js server, with ```bodyParser```
We also swapped out ```mysql``` with ```promise-mysql``` for better compatability with Typescript.

## Models

Relevant to this post we have

### Photo
```javascript
class PhotoEntity {
  public ID: number;
  public URL: string;
  public OWNER: number;
}
```
### Tag
```javascript
class TagEntity {
  public ID: number;
  public LABEL: string;
}
```
### PhotoTag
```javascript
class PhotoTagEntity {
  public ID: number;
  public PHOTOID: number;
  public TAGID: number;
}
```

## Controller

Same as the last post, we have our ```/photos``` endpoint that returns a list of photos, each with a list of tags associated with the photos. 

```javascript
const getAllPhotos = (req: Request, res: Response) => {
  photoManager
    .getAllPhotos()
    .then((views) => res.status(200).send(views))
    .catch((err) => res.status(500).send({ error: true, message: err }));
};
```

Nothing new here. This is the same as its always been. But the fun comes in the next layer...

## Manager

My plan is still the same:

    1. Grab the photos
    2. Grab the PhotoTag entries for each photo 
    3. Extract the tag IDs from the PhotoTag entity for each photo 
    4. Grab the tags with the extracted tag IDs 
    5. Attach the resulting tags to the photo view model and add the view model to a list.

I was running into an issue when trying to clean up my managers. I needed a way to pass data along between the different steps. What I was doing before was using a combination of callbacks and closures to access data in higher scopes. However, I opted for a more functional approach.

```javascript
class PhotoManager {
  public getAllPhotos = () => {
    return new Promise<PhotoViewModel[]>((resolve, reject) => {
      photoRepository
        .getAllPhotos()
        .then(getRelatedPhotoData)
        .then(convertAllDataToViews)
        .then(aggregateDataIntoPhotoViews)
        .then(resolve)
        .catch(reject);
    });
  }

  // other methods
}
```

I was able to simplify my manager layer a lot by taking advantage of a neat property of Promises. Because we are able to chain Promises using ```.then```, we can pass along data that we need by returning it wrapped in a Promise. This allows us to utilize 'zero point' function calls, and results in a much cleaner manager. If we look at our ```getRelatedPhotoData``` method:

```javascript
const getRelatedPhotoData = (entities: PhotoEntity[]) => {
  const data = entities.map((photoEntity) => {
    return Promise.all([
      photoEntity,
      photoTagRepository.getPhotoTagsForPhotoId(photoEntity.ID)
    ]);
  });
  return Promise.all(data);
};
```

```entities``` is passed in as the result of the ```photoRepository.getAllPhotos()``` call. This contacts the DB and returns an array of Photo entities. What we also do is we call a method from the PhotoTag repository that returns a list of PhotoTags for a particular Photo ID. We are grouping them as a tuple and Promisifying the list of tuples. This list of tuples will be fed into the next function, which will perform some operation and return a Promise of that transformed data. This continues until we ```resolve``` with our data, which signals to the caller that the Promise has finished executing and returned some result.

___
> Woah...
___

This let me write cleaner, more modular code. And it had the added bonus of making me feel like a badass 😎

Here are the rest of the methods in the manager so you can see how data flows across multiple methods.

```javascript
const convertAllDataToViews = (
  data: Array<[PhotoEntity, PhotoTagEntity[]]>
) => {
  const views = data.map(([photoEntity, photoTagEntities]) => {
    return Promise.all([
      PhotoViewModel.fromEntity(photoEntity),
      tagManager.getTagsById(photoTagEntities.map((entity) => entity.TAGID))
    ]);
  });
  return Promise.all(views);
};

const aggregateDataIntoPhotoViews = (
  views: Array<[PhotoViewModel, TagViewModel[]]>
) => {
  const aggregatedViews = views.map(([photoView, tagViews]) => {
    photoView.tags = tagViews;
    return photoView;
  });
  return Promise.resolve(aggregatedViews);
};
```

## Repository

For completeness sake, here is the Photo repository:

```javascript
class PhotoRepository {
  public getAllPhotos = () => {
    return new Promise<PhotoEntity[]>((resolve, reject) => {
      return db
        .getConnection()
        .then((connection) => {
          const query = connection.query({
            sql: "SELECT * FROM PHOTO"
          });
          return Promise.all([connection, query]);
        })
        .then(releaseAndEndConnection)
        .then((results) => resolve((results as PhotoEntity[]) || []))
        .catch((err) => reject(`${err}`));
    });
  }

  // other methods
}
```

As you can see, I utilized the same methodology in chaining promises at the repository level as well. 

I learned a lot about Promises and hopefully you did too! Make sure to let me know in the comments below. 