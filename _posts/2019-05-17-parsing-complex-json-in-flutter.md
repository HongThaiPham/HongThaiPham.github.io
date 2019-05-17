---
layout: post
title: "Parsing complex JSON in Flutter"
date: 2019-05-17 21:14:00 +0700
categories: [flutter]
tags: [dart, flutter, json parsing]
image: parsing-complex-json-in-flutter.jpeg
---

I have to admit, I was missing the **gson** world of Android after working with JSON in Flutter/Dart. When I started working with APIs in Flutter, JSON parsing really had me struggle a lot. And I’m certain, it confuses a lot of you beginners.

We will be using the built in _dart:convert_ library for this blog. This is the most basic parsing method and it is only recommended if you are starting with Flutter or you’re building a small project. Nevertheless, knowing the basics of JSON parsing in Flutter is pretty important. When you’re good at this, or if you need to work with a larger project, consider code generator libraries like [json_serializable](https://pub.dartlang.org/packages/json_serializable), etc. If possible, I will discover them in the future articles.

Fork this [sample project](https://github.com/PoojaB26/ParsingJSON-Flutter). It has all the code for this blog that you can experiment with.

## JSON structure #1 : Simple map

Let’s start with a simple JSON structure from [student.json](https://github.com/PoojaB26/ParsingJSON-Flutter/blob/master/assets/student.json)

```json
{
  "id": "487349",
  "name": "Pooja Bhaumik",
  "score": 1000
}
```

**Rule #1 : Identify the structure. Json strings will either have a Map (key-value pairs) or a List of Maps.**

**Rule #2 : Begins with curly braces? It’s a map.
Begins with a Square bracket? That’s a List of maps.**

_student.json_ is clearly a map. ( E.g like, _id_ is a key, and _487349_ is the value for _id_)

Let’s make a PODO (Plain Old Dart Object?) file for this json structure. You can find this code in [student_model.dart](https://github.com/PoojaB26/ParsingJSON-Flutter/blob/master/lib/model/student_model.dart) in the sample project.

```dart
class Student{
  String studentId;
  String studentName;
  int studentScores;

  Student({
    this.studentId,
    this.studentName,
    this.studentScores
 });
}
```

Perfect!

Was it? Because there was no mapping between the json maps and this PODO file. Even the entity names don’t match.

I know, I know. We are not done yet. We have to do the work of mapping these class members to the json object. For that, we need to create a factory method. According to Dart documentation, we use the factory keyword when implementing a constructor that doesn’t always create a new instance of its class and that’s what we need right now.

```dart
factory Student.fromJson(Map<String, dynamic> parsedJson){
    return Student(
      studentId: parsedJson['id'],
      studentName : parsedJson['name'],
      studentScores : parsedJson ['score']
    );
}
```

Here, we are creating a factory method called Student.fromJson whose objective is to simply deserialize your json.

**I’m a little noob, can you tell me about Deserialization?**

Sure. Let’s tell you about **Serialization** and **Deserialization** first. **Serialization** simply means writing the data(which might be in an object) as a string, and **Deserialization** is the opposite of that. It takes the raw data and reconstructs the object model. In this article, we mostly will be dealing with the deserialization part. In this first part, we are deserializing the json string from _student.json_

So our factory method could be called as our converter method.

Also must notice the parameter in the _fromJson_ method. It’s a _Map<String, dynamic>_ It means it maps a _String_ key with a _dynamic_ value. That’s exactly why we need to identify the structure. If this json structure were a List of maps, then this parameter would have been different.

**But why dynamic?**

Let’s look at another json structure first to answer your question.

![](https://cdn-images-1.medium.com/max/800/1*aYehHPUoXS4S-CVLWg1NCQ.png)

_name_ is a Map<String, String> ,_majors_ is a Map of String and List<String> and _subjects_ is a Map of String and List<Object>

Since the key is always a _string_ and the value can be of any type, we keep it as _dynamic_ to be on the safe side.

Check the full code for _student_model.dart_ [here](https://github.com/PoojaB26/ParsingJSON-Flutter/blob/master/lib/model/student_model.dart).

## Accessing the object

Let’s write _student_services.dart_ which will have the code to call _Student.fromJson_ and retrieve the values from the _Student_ object.

**Snippet #1 : imports**

```dart
import 'dart:async' show Future;
import 'package:flutter/services.dart' show rootBundle;
import 'dart:convert';
import 'package:flutter_json/student_model.dart';
```

The last import will be the name of your model file.

**Snippet #2 : load Json Asset (optional)**

```dart
Future<String> _loadAStudentAsset() async {
  return await rootBundle.loadString('assets/student.json');
}
```

In this particular project, we have our json files in the assets folder, so we have to load the json in this way. But if you have your json file on the cloud, you can do a network call instead. _Network calls are out of the scope of this article._

**Snippet #3 : load the response**

```dart
Future loadStudent() async {
  String jsonString = await _loadAStudentAsset();
  final jsonResponse = json.decode(jsonString);
  Student student = new Student.fromJson(jsonResponse);
  print(student.studentScores);
}
```

In this _loadStudent()_ method,
Line 1 : loading the raw json String from the assets.
Line 2 : Decoding this raw json String we got.
Line 3 : And now we are deserializing the decoded json response by calling the _Student.fromJson_ method so that we can now use _Student_ object to access our entities.
Line 4 : Like we did here, where we printed _studentScores_ from _Student_ class.

_Check your Flutter console to see all your print values. (In Android Studio, its under Run tab)_

And voila! You just did your first JSON parsing (or not).

_Note: Remember the 3 snippets here, we will be using it for the next set of json parsing (only changing the filenames and method names), and I won’t be repeating the code again here. But you can find everything in the sample project anyway._

## JSON structure #2 : Simple structure with arrays

Now we conquer a json structure that is similar to the one above, but instead of just single values, it might also have an array of values.

```json
{
  "city": "Mumbai",
  "streets": ["address1", "address2"]
}
```

So in this _address.json_, we have _city_ entity that has a simple _String_ value, but _streets_ is an array of _String_.

As far as i know, Dart doesn’t have an array data type, but instead has a _List<datatype>_ so here _streets_ will be a _List<String>_.

Now we have to check **Rule#1** and **Rule#2** . This is definitely a map since this starts with a curly brace. _streets_ is still a _List_ though, but we will worry about that later.

So the _address_model.dart_ initially will look like this

```dart
class Address {
  final String city;
  final List<String> streets;

  Address({
    this.city,
    this.streets
  });
}
```

Now since this is a map, our _Address.fromJson_ method will still have a _Map<String, dynamic>_ parameter.

```dart
factory Address.fromJson(Map<String, dynamic> parsedJson) {

  return new Address(
      city: parsedJson['city'],
      streets: parsedJson['streets'],
  );
}
```

Now construct the _address_services.dart_ by adding the 3 snippets we mentioned above. Must remember to put the proper file names and method names. Sample project already has _address_services.dart_ constructed for you.

Now when you run this, you will get a nice little error. :/

```dart
type 'List<dynamic>' is not a subtype of type 'List<String>'
```

I tell you, these errors have come in almost every step of my development with Dart. And you will have them too. So let me explain what this means. We are requesting a _List<String>_ but we are getting a _List<dynamic>_ because our application cannot identify the type yet.

So we have to explicitly convert this to a _List<String>_

```dart
var streetsFromJson = parsedJson['streets'];
List<String> streetsList = new List<String>.from(streetsFromJson);
```

Here, first we are mapping our variable _streetsFromJson_ to the _streets_ entity. _streetsFromJson_ is still a _List<dynamic>_. Now we explicitly create a new _List<String>_ streetsList that contains all elements from _streetsFromJson_.

Check the updated method [here](https://github.com/PoojaB26/ParsingJSON-Flutter/blob/master/lib/model/address_model.dart). Notice the return statement now.

Now you can run this with _address_services.dart_ and this will work perfectly.

## Json structure #3 : Simple Nested structures

Now what if we have a nested structure like this from [shape.json](https://github.com/PoojaB26/ParsingJSON-Flutter/blob/master/assets/shape.json)

```json
{
  "shape_name": "rectangle",
  "property": {
    "width": 5.0,
    "breadth": 10.0
  }
}
```

Here, _property_ contains an object instead of a basic primitive data-type.

_So how will the PODO look like?_

Okay, let’s break down a little.

In our _shape_model.dart_ , let’s make a class for _Property_ first.

```dart
class Property{
  double width;
  double breadth;

  Property({
    this.width,
    this.breadth
    });
}
```

Now let’s construct the class for _Shape_. I am keeping both classes in the same Dart file.

```dart
class Shape{
  String shapeName;
  Property property;

  Shape({
    this.shapeName,
    this.property
  });
}
```

Notice how the second data member _property_ is basically an object of our previous class _Property_.

**Rule #3: For nested structures, make the classes and constructors first, and then add the factory methods from bottom level.**

By bottom level, we mean, first we conquer _Property_ class, and then we go one level above to the _Shape_ class. This is just my suggestion, not a Flutter rule.

```dart
factory Property.fromJson(Map<String, dynamic> json){
  return Property(
    width: json['width'],
    breadth: json['breadth']
  );
}
```

This was a simple map.

But for our factory method at _Shape_ class, we cant just do this.

```dart
factory Shape.fromJson(Map<String, dynamic> parsedJson){
  return Shape(
    shapeName: parsedJson['shape_name'],
    property : parsedJson['property']
  );
}
```

_property : parsedJson['property']_ First, this will throw the type mismatch error —

```dart
type '_InternalLinkedHashMap<String, dynamic>' is not a subtype of type 'Property'
```

And second, _hey we just made this nice little class for Property, I don’t see it’s usage anywhere._

Right. We must map our Property class here.

```dart
factory Shape.fromJson(Map<String, dynamic> parsedJson){
  return Shape(
    shapeName: parsedJson['shape_name'],
    property: Property.fromJson(parsedJson['property'])
  );
}
```

So basically, we are calling the _Property.fromJson_ method from our _Property_ class and whatever we get in return, we map it to the _property_ entity. Simple! Check out the code here.

Run this with your _shape_services.dart_ and you are good to go.

## JSON structure #4 : Nested structures with Lists

Let’s check our [product.json](https://github.com/PoojaB26/ParsingJSON-Flutter/blob/master/assets/product.json)

```json
{
  "id": 1,
  "name": "ProductName",
  "images": [
    {
      "id": 11,
      "imageName": "xCh-rhy"
    },
    {
      "id": 31,
      "imageName": "fjs-eun"
    }
  ]
}
```

Okay, now we are getting deeper. I see a list of objects somewhere inside. Woah.

Yes, so this structure has a List of objects, but itself is still a map. (Refer **Rule #1**, and **Rule #2**) . Now referring to **Rule #3**, let’s construct our _product_model.dart_.

So we create two new classes _Product_ and _Image_.
Note: _Product_ will have a data member that is a List of _Image_

```dart
class Product {
  final int id;
  final String name;
  final List<Image> images;

  Product({this.id, this.name, this.images});
}

class Image {
  final int imageId;
  final String imageName;

  Image({this.imageId, this.imageName});
}
```

The factory method for _Image_ will be quite simple and basic.

```dart
factory Image.fromJson(Map<String, dynamic> parsedJson){
 return Image(
   imageId:parsedJson['id'],
   imageName:parsedJson['imageName']
 );
}
```

Now for the factory method for _Product_

```dart
factory Product.fromJson(Map<String, dynamic> parsedJson){

  return Product(
    id: parsedJson['id'],
    name: parsedJson['name'],
    images: parsedJson['images']
  );
}
```

This will obviously throw a runtime error

```dart
type 'List<dynamic>' is not a subtype of type 'List<Image>'
```

And if we do this,

```dart
images: Image.fromJson(parsedJson['images'])
```

This is also definitely wrong, and it will throw you an error right away because you cannot assign an _Image_ object to a _List<Image>_

So we have to create a _List<Image>_ and then assign it to _images_

```dart
var list = parsedJson['images'] as List;
print(list.runtimeType); //returns List<dynamic>
List<Image> imagesList = list.map((i) => Image.fromJson(i)).toList();
```

list here is a _List<dynamic>_. Now we iterate over the list and map each object in list to _Image_ by calling _Image.fromJson_ and then we put each map object into a new list with _toList()_ and store it in _List<Image> imagesList_. Find the full code [here](https://github.com/PoojaB26/ParsingJSON-Flutter/blob/master/lib/model/product_model.dart).

## JSON structure #5 : List of maps

Now let’s head over to [photo.json](https://github.com/PoojaB26/ParsingJSON-Flutter/blob/master/assets/photo.json)

```json
[
  {
    "albumId": 1,
    "id": 1,
    "title": "accusamus beatae ad facilis cum similique qui sunt",
    "url": "http://placehold.it/600/92c952",
    "thumbnailUrl": "http://placehold.it/150/92c952"
  },
  {
    "albumId": 1,
    "id": 2,
    "title": "reprehenderit est deserunt velit ipsam",
    "url": "http://placehold.it/600/771796",
    "thumbnailUrl": "http://placehold.it/150/771796"
  },
  {
    "albumId": 1,
    "id": 3,
    "title": "officia porro iure quia iusto qui ipsa ut modi",
    "url": "http://placehold.it/600/24f355",
    "thumbnailUrl": "http://placehold.it/150/24f355"
  }
]
```

Uh, oh. **Rule #1** and **Rule #2** tells me this can’t be a map because the json string starts with a square bracket. So this is a **List of objects**? Yes. The object being here is _Photo_ (or whatever you’d like to call it).

```dart
class Photo{
  final String id;
  final String title;
  final String url;

  Photo({
    this.id,
    this.url,
    this.title
}) ;

  factory Photo.fromJson(Map<String, dynamic> json){
    return new Photo(
      id: json['id'].toString(),
      title: json['title'],
      url: json['json'],
    );
  }
}
```

But its a list of _Photo_ , so does this mean you have to build a class that contains a _List<Photo>_?

Yes, I would suggest that.

```dart
class PhotosList {
  final List<Photo> photos;

  PhotosList({
    this.photos,
  });
}
```

Also notice, this json string is a List of maps. So, in our factory method, we won’t have a _Map<String, dynamic>_ parameter, because it’s a List. And that is exactly why it’s important to identify the structure first. So our new parameter would be a _List<dynamic>_.

```dart
factory PhotosList.fromJson(List<dynamic> parsedJson) {

    List<Photo> photos = new List<Photo>();

    return new PhotosList(
       photos: photos,
    );
  }
```

This would throw an error

```dart
Invalid value: Valid value range is empty: 0
```

Hey, because we never could use the _Photo.fromJson_ method.
What if we add this line of code after our list initialization?

```dart
photos = parsedJson.map((i)=>Photo.fromJson(i)).toList();
```

Same concept as earlier, we just don’t have to map this to any key from the json string, because it’s a List, not a map. Code [here](https://github.com/PoojaB26/ParsingJSON-Flutter/blob/master/lib/model/photo_model.dart).

## JSON structure #6 : Complex nested structures

Here is [page.json](https://github.com/PoojaB26/ParsingJSON-Flutter/blob/master/assets/page.json).

I will request you to solve this. It is already included in the sample project. You just have to build the model and services file for this. But I won’t conclude before giving you hints and tips (if case, you need any).

**Rule#1** and **Rule#2** as usual applies. Identify the structure first. Here it is a map. So all the json structures from 1–5 will help.

**Rule #3** asks you to make the classes and constructors first, and then add the factory methods from bottom level. Just another tip. Also add the classes from the deep/bottom level. For e.g, for this json structure, make the class for _Image_ first, then _Data_ and _Author_ and then the main class _Page_. And add the factory methods also in the same sequence.

For class Image and Data refer to **Json structure #4**.

For class Author refer to **Json structure #3**

Beginner’s tip: While experimenting with any new assets, remember to declare it in the \_pubspec.yaml \_file.

And that’s it for this Fluttery article. This article may not be the best JSON parsing article out there, (because I’m still learning a lot) but I hope it got you started.

[Github](https://medium.com/@carlosAmillan/parseando-json-complejo-en-flutter-18d46c0eb045)

Source: [Parsing complex JSON in Flutter](https://medium.com/flutter-community/parsing-complex-json-in-flutter-747c46655f51)

-- Thanks Pooja Bhaumik
