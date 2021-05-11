# C# Live Project

## Introduction
This work was completed over two week internship with Prosper IT Conculting beginning on 4/19/21. I worked with a team of peers in developing a full-scale MVC Web Application. We worked within the Scrum Framework, utilizing Azure DevOps for flexible version control and task-management while developing new features for our build. 

Our project is a content management system(CMS) for a thearte company. I chose to tackle the CastMember area; creating a database with Create-Read-Update-Delete(CRUD) functionalities for the theatre's Cast Members. I completed various front-end and back-end designs to meet our user-oriented goals, ending up with a function and feel I am proud of. Over the two-week sprint I also had the opportunity to work with project-management and team programming skils that I know I will use again on future projects.

## Sections

- [Back End](#Back-End)
- [Front End](#Front-End)
- [Retrospective](#Retrospective)

## Back End

  
### Code-First DataBase Model

The first 'story' in this are was to create a basic CastMember Model. We used Entity Framework's Code First DataBase capabilities to generate our database. Here are the basic properties of a CastMember that were relevant to our users. 

```
  public class CastMember
    { 
        [Key]
        public int CastMemberId { get; set; }
        [Required]
        public string Name { get; set; }
        public int? YearJoined { get; set; }
        [Required]
        public PositionEnum MainRole { get; set; }
        [Required]
        public string Bio { get; set; }
        [Required]
        public bool CurrentMember { get; set; }
        [Required]
        public string Character { get; set; }
        public int? CastYearLeft { get; set; }
        public int? DebutYear { get; set; } 
 ```
### Photo Upload 

For this next story, I was tasked with adding Photo upload capabilities. At first, I used a single property (CastMember.photo) to store and present the photo, attempting to convert it as it was uploaded, but this didn't work. By sketching out the function, I ralized I needed a property to store the volatile file, and another to save its converted form. The file can be stored temporarily in a HttpPostedFileBase wrapper, but must be converted to a byte-array in order to store it permanatly in the DataBase. Therefor I created two properties in the CastMember Model to store each form of a photo. 
```
[NotMapped]
        public HttpPostedFileBase File { get; set; }   //Photo Input/Upload/Display
        public byte[] Photo { get; set; }              //Savable Storage
```

Next, I created functions to convert the file to bytes, and vise versa. This alows Photo storage and display.

```
//Photo Storage and Retrieval 
        public byte[] FileToBytes(HttpPostedFileBase file)
        {
            byte[] bytes;
            using (BinaryReader br = new BinaryReader(file.InputStream))       
                bytes = br.ReadBytes(file.ContentLength);                 //Convert to Byte array
            return bytes;
        }

        public ActionResult GetImage(int id)
        {
            byte[] bytes = db.CastMembers.Find(id).Photo;     //Find Bytes
            if (bytes == null) return null;
            return File(bytes, "image/jpg");                  //Convert to image-file
        }
```

  
### RDBM: Many CastMembers-to-Many Productions
My last back-end task was to build a Many-to-Many link between the CastMember and the Production Models. Entity Framework creates a hidden join-table to accomodate each pairing of a CastMember and a Production. Accessing and saving this the data (castM.Prod / Prod.castMem) requires special treatment. I delt with a lot of NullReferenceExceptions before realing that these property-list properties were volatile across page-loads. I needed proxie properties for selection, storage and title-diplay!
```

        //Productions
        
        public IEnumerable<SelectListItem> ProductionsListItems { get; set; }   //List Box List-Items
        [Required]
        public IEnumerable<string> SelectedProductions { get; set; }            //Underlying Value (ProdId) as string.        

        public virtual HashSet<Production> Productions { get; set; }            //Stores Objects Resulting from SelectedProductions Query
        public CastMember() { this.Productions = new HashSet<Production>(); }   //Property Constructor for EF Nav.
        public List<string> ProductionTitles { get; set; }                      //Pulls title string from Prod objects for display.
    }
```



The EF statements that manipulate the hidden join table must be created and populated from the database in a single instance. One cannot use the CastMember object to store this information across pages. After I realized that fact I created a listbox with proxy 'production' items that could be used to query the db when associating CastMembers and Productions. 
```
// Return list of Productions to populate dropdown-list form-input
        public List<SelectListItem> GetProductionList()
        {
            var productions = new List<SelectListItem>();

            foreach (Production production in db.Productions)
            {
                var item = new SelectListItem
                {
                    Text = production.Title,
                    Value = production.ProductionId.ToString(),
                    Selected = production.IsSelected
                };
                productions.Add(item);
            }
            return productions;
        }
```
* For the Create Method, I had the user select productions from a list box. (Covered in FrontEnd) This produces a list of numbers as strings, used to call the db.Producitons
```
     [HttpPost]
        [ValidateAntiForgeryToken]
        public ActionResult Create(IEnumerable<string> selectedProductions, [Bind(Include = "CastMemberID,Name,YearJoined,ProductionTitles,MainRole,Bio,CurrentMember,Character,CastYearLeft,DebutYear, File")] CastMember castMember)
        {     
            if (ModelState.IsValid)
            {
                if (selectedProductions == null)             //Production ListBox Null Protection; needs improvement.
                {
                    return RedirectToAction("Create");
                }

                castMember.SelectedProductions = selectedProductions;        //Save Production selection to model.
                foreach(string selection in castMember.SelectedProductions)  //Then populate CastMember.Productions with db.Productions
                {
                    int selectionInt = Int32.Parse(selection);
                    castMember.Productions.Add(db.Productions.Where(p => p.ProductionId == selectionInt).SingleOrDefault());
                }

                castMember.Photo = FileToBytes(castMember.File);           //Convert Uploaded photo file to Byte[]
                
                db.CastMembers.Add(castMember);
                db.SaveChanges();
                return RedirectToAction("Index");
            }

            return View(castMember);
        }
  
```
* When accessing the Edit all list properties related to CastMember.Productions are wiped. New selections cannot be save with EntityState.Modified. First I save the Non-Nav Properties with EntityState.Modified
```
        [HttpPost]
        [ValidateAntiForgeryToken]
        public ActionResult Edit(IEnumerable<string> selectedProductions, CastMember castMember )
        {
            if (ModelState.IsValid)
            {
                // Photo Upload
                if (castMember.File != null)
                    castMember.Photo = FileToBytes(castMember.File);

                db.Entry(castMember).State = EntityState.Modified;    //Save non-Nav properties to db before altering Nav properties. 
                db.SaveChanges();
```
* Then I repopulated existing selections from db. This allows a user to Edit a property without having to reselect Productions. 
```

                if (selectedProductions == null){castMember.SelectedProductions = GetSelectedProductions(castMember.CastMemberId); }        //Production ListBox Null maintains original selections.
```
* Then process any new selections and save changes. 
```
                var productions = db.Productions.ToHashSet<Production>();                                                                   //Create Productions List   
                castMember = db.CastMembers.Include(x => x.Productions).FirstOrDefault(x => x.CastMemberId == castMember.CastMemberId);     //Repopulate castMember from db. Needed for EF change-tracking
                castMember.Productions.Clear();                                                                                             //Clear old Producitons
                foreach (string selection in castMember.SelectedProductions)                                                                //Add new Production for each selection
                {
                    int selectionInt = Int32.Parse(selection);
                    castMember.Productions.Add(productions.Where(p => p.ProductionId == selectionInt).SingleOrDefault());
                }
                db.SaveChanges();
                return RedirectToAction("Index");
            }
            return View(castMember);            
        }
```



## Front-End

  
### Index
The Theatre Virtigo CastMembers Index Page Shows everyone involved in a production. With stylish and lightweight controls, managing this list of CastMembers - or reading more about them - is dependable and enjoyable.
![Index](./images/CastMemberIndex.gif)



On loading the Index Page, my code creates rows for each Produciton. Then it creates a card for each CastMember in that a Production, displaying them in that productions' row. CastMembers can be a part of more than one produciton and the page actively reflects this data-structure. By duplicating CastMember cards for each production they are in, we can easily see how the Theatre group is organized. 
```@{
    foreach (var title in ViewBag.ProductionTitles)    //For Each Distinct ProductionTitle...
    {
      <div>
        <div class="prod-form-label">@Html.DisplayFor(modelItem => title)</div> <!--Display Title-->
        <hr class="cms-bg-main-light mt-2 pb-2" />                              <!--And Divider Line-->
      </div>
      <div class="row mb-2"> 
        @foreach (var castMember in Model)   //For Each CastMember of that Production, Display Card.
        {
           List<string> titlesList = castMember.Productions.Select(x => x.Title).ToList();  
           if (titlesList.Contains(title)) 
           { 
             <div class="card bg-dark m-sm-auto w-25">            
                <img src="data:image;base64,@System.Convert.ToBase64String(castMember.Photo)" class=" card-img-top"> <!--CastMember Card Image-->

                <div class="prod-overlay text-center">                                                               <!--Overlay link to Details-->
                  @Html.ActionLink("Details", "Details", null, new { id = castMember.CastMemberId },
                  new { @class = "detailsLink fas fa-edit" })

                  <div class=" prod-overlay-icon-container text-center ">                                            <!--Overlay Icon Buttons-->
                    <a href=@Url.Action("Edit", "CastMembers", new { id = castMember.CastMemberId } )
                       title="Edit"><i class="far fa-edit fa-2x prod-icon mr-2"></i></a>
                    <a href=@Url.Action("Delete", "CastMembers", new { id = castMember.CastMemberId } )
                       title="Delete"><i class="fas fa-user-times fa-2x prod-icon ml-2"></i></a>
                  </div>
                </div> 

                <!--CastMember Info-->
                <div class="card-body bg-dark text-light">
                  <p class="prod-form-label">@Html.DisplayFor(modelItem => castMember.Name)</p>
                  <hr class="cms-bg-secondary mt-1 pb-1" />
                  @Html.DisplayFor(modelItem => castMember.MainRole)<br />

                  <!--Display Number of Productions each CastMember is a part of.-->
                  @if (castMember.Productions.Count > 1)
                  {<p>Member of:<br /> @castMember.Productions.Count Productions</p><!--Singular-->}
                  else {<p>Member of:<br /> @castMember.Productions.Count Production</p> <!--Plural-->}
                </div>
          </div>
          }
        }
      </div>
  }
}
```
<br>
<img src="images/Screenshot%202021-04-12%20155131.png" alt="DogActors" width="350" /></br>
<sub>Testing with Dog Actors was a good time.</sub>

---

### Create/Edit

Tidy and stylish form-inputs and a photo display make adding a new Castmember an easy task. On these pages we see a list-box for selecting one or more productions. This list-box is populated with ListBoxItems created from the db and passed in via viewbag. 
  
When selected, the list-box outputs a string-list of the productions IDs which can be used in a db call to populate CastMember objects with a list of Produciton objects. 



  
![Production-Select](./images/ProdSelect.gif)
<sub>Multi-select with Ctrl-lClick.</sub>


#### If no selection is re-entered when Editing, original production selections are maintained. 
![Edit](/images/EditCastMember.gif)

- [See Code Above](#rdbm-many-castmembers-to-many-productions)

---

### Details/Delete
The Details and Delete pages are uniquely structured to highligh the CsatMember's photo and their Bio. Other Info is displayed balow, along with links to other pages . 
  
![Delete](images/Delete.gif)



  
## Retrospective
  ### Challenges
 

  Building a many-to-many link with Entity Framework proved to be my biggest challenge on this project. Hours of mystrious nullReferenceExceptions. But with unit testing, study and communication with my team-members and mentors I was able to navigate the assumptions behind EntityFrameworks interpretation of my code. After creating special handling for displaying, selecting, and handling Production-CastMember associations I was able to create what was needed. After cleaning it up, I am proud of this contribution to the project.

  ### Successes 

  Im very happy with the overall user-experience of my work. My index page is reactively structured around DB entities; clearly showing their relationships. User-controls are strightforward, and contained in a polised asthetic I helped create to enhance the pre-existing look and feel of the site. 

  Throughout this Project I worked as a coachable team-member, fell in love with Azure DevOps and learned how to learn on the job. I look ahead to future builds and more excellent collaborations.   
