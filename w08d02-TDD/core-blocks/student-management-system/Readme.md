# Student Management System

## Introduction
In this web app you'll be building a SOAL student profile showcasing each student profile on respective `/student/:id`. In this challenge you have to write a simple RESTful API with Node.js. You have to test CRUD operations on SOAL

## Release 0: Data Definition
- Student schema should have:
	- Name
	- Technical Skills
	- Education Background(You can get creative with this. Add whatever college you want)
	- Previous Experience with Designations(Be creative with this at Google, Microsoft)
	- Join date on SOAL
	- Age
	- Social or !Social(Boolean) [this `isSocial` field depends on how the person interacts with other team members]
	- User photos(user your phone camera or social media)

## Release 1: CRUD
- Write GET `/student` route to retrive all students
- write POST `/student` route save a new student
- write GET `/student/:id` route to retrieve a student given its id.
- write DELETE `/student/:id` route to delete a student given its id.
- write PUT `/student/:id` to update a student given its id

## Release 2: Postman
- Start by testing all these above routes with POSTMAN
- Take screenshots of Postman API check and push them to Github along with your code

## Release 3: Writing Proper Tests using Chai & Mocha
- Check all the `POST` and `Get Routes` using Chai & Mocha like you learned in this [application](https://scotch.io/tutorials/test-a-node-restful-api-with-mocha-and-chai#toc-bonus-mockgoose)