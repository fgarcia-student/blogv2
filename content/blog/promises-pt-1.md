---
title: "Promises Pt 1"
date: 2019-04-26T23:16:00-04:00
draft: false
---

## Promises have their own kind of hell. And it's almost as crazy as callback hell.

Writing DAO and Manager layers in Node.js with promises has been some adventure. I'm going to talk through a simple get request for some photos with tags and demonstrate my process and pain points. Hopefully any reader (or myself) in the future can smack me in the face and say 

> You dope! Just do [fill in better solution here]

Here we go A GET request comes through the /photos controller and triggers a method call. This method is called getAllPhotos(). 

```javascript
const getAllPhotos = (req: Request, res: Response) => {
  photoManager
    .getAllPhotos()
    .then((views) => res.status(200).send(views))
    .catch((err) => res.status(500).send({ error: true, message: err }));
};
```

Now, because we are in a Node environment, all of our DB communication must be asynchronous. In a Node environment, this is handled in one of three ways (really only 2 AFAIK):

    1. Callbacks
    2. Promises
    3. async / await (still uses promises under the hood)
    
Now because we are using MySQL as our database, we have the ```mysql``` npm package installed for our database communication. By default this library uses callbacks to handle asynchronous actions. This can be changed using the ```util.promisify``` function off of the ```util``` package that you get when working in a Node environment.

So with that being said we can move on to the manager for this entity. the manager returns a Promise which will resolve at some point. This is the key to asynchronous development in Node.js using promises.

> The method, if it requires async work, must return a Promise. Business logic can be placed within the promise and when a result is recieved, it can be passed into the resolve method exposed by the Promise.

So this basically is saying everything past the controller has to return a promise. If not, it can't have any async method calls otherwise they will not work as expected. So within our promise, we call a function getAllPhotos() from the repository. This returns a promise to some entities. I know im jumping around but here are our entities.

```javascript
class PhotoEntity {
  public ID: number;
  public URL: string;
  public OWNER: number;
}

class TagEntity {
  public ID: number;
  public LABEL: string;
}

class PhotoTagEntity {
  public ID: number;
  public PHOTOID: number;
  public TAGID: number;
}
```

So here is our plan:

    1. Grab the photos
    2. Grab the PhotoTag entries for each photo 
    3. Extract the tag IDs from the PhotoTag entity for each photo 
    4. Grab the tags with the extracted tag IDs 
    5. Attach the resulting tags to the photo view model and add the view model to a list.
    
This series of operations results in code resembling a level from Mario Maker. No joke. And to signify the end of this entire operation we have to compare the lengths of the view model list and entity list, and once they are equal we know to resolve the view models. There has to be a better way! If you guys know of one please let me know. This results in a pretty quick query but I know there is room for improvement. As I continue to grow and learn more about promises and asynchronous programming I will probably look back at this code and cringe.

Stay coding everyone ✌️