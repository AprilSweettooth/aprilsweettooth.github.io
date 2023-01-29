---
layout: page
title: contact
---

Contact me [here](sqhy2@cam.ac.uk)

<br><br>


<form accept-charset="UTF-8" action="https://getform.io/f/5b4d8843-e896-496f-a77d-56aeb82d92ce" method="POST" enctype="multipart/form-data" target="_blank">
          <div class="form-group">
            <label for="exampleInputEmail1" required="required">Email address</label>
            <input type="email" name="email" class="form-control" id="exampleInputEmail1" aria-describedby="emailHelp" placeholder="Enter email">
          </div>
          <div class="form-group">
            <label for="exampleInputName">Name</label>
            <input type="text" name="name" class="form-control" id="exampleInputName" placeholder="Enter your name" required="required">
          </div>
          <div class="form-group">
            <label for="exampleFormControlSelect1">Favourite Platform</label>
            <select class="form-control" id="exampleFormControlSelect1" name="platform" required="required">
              <option>Github</option>
              <option>Gitlab</option>
              <option>Bitbucket</option>
            </select>
          </div>
          <hr>
          <div class="form-group mt-3">
            <label class="mr-2">Upload your CV:</label>
            <input type="file" name="file">
          </div>
          <hr>
          <button type="submit" class="btn btn-primary">Submit</button>
        </form>

{% include logos.html %}

