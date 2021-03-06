# Room Database Prototype - Android
##### References:
- Tutorial: The Comprehensive Android Development Masterclass - Section 28
- Codelab: Android Room with a View - Java

## Purpose: 
The purpose of this app is to explore the Room Library and how it provides a layer of abstraction over SQLite to allow database access. Specifically, I am wanting to explore the 'convenience annotations' that minimise error-prone code (SQL). The documentation highly recommendeds that you Room instead of using the SQLite APIs directly. 

##### Room Documentation: https://developer.android.com/training/data-storage/room?gclid=Cj0KCQiAlsv_BRDtARIsAHMGVSapV4bg9VhEDDfKxyhK2fuLiPLOf8n7JzNkSMuzUBRrEcU-T2The1gaAsceEALw_wcB&gclsrc=aw.ds


## Room Architecture:
![find](/RoomModel.png)

##### Source: https://developer.android.com/codelabs/android-room-with-a-view#1

## Setup:
Check the documentation for the dependencies you need to add to your app's build.gradle file. You may need to revisit the version number every now and then.

## Primary Components:

### Data Entities
#### @Entity
Use this annotation in your model to let the compiler know that you want the model to be an entity. An entity is an annotated class that describes a database table when working with Room. It is similar to the Entities in CoreData when working with the Apple SDKs, however, the primary difference being the annotations within the Model. Room will use this class  to both create a table and instantiate objects from rows in the database.

```
@Entity(tableName = "contact_table")
public class Contact {
```

#### @PrimaryKey(autoGenerate = true)
This annotation lets the compiler know that we want the id property to also be a primary key. As we have declared the key to be auto generated, we can remove id from the constructor.

```
@PrimaryKey(autoGenerate = true)
    private int id;
    
    
    //As id is a PK, it will be automatically generate, therefore, it doesn't need to go into the constructor.
    public Contact(@NonNull String name, @NonNull String occupation) {
        this.name = name;
        this.occupation = occupation;
    }
```

#### @NonNul 
This cannotation can be used in the constructor to let the compiler know that these properties must have a value.

```
public Contact(@NonNull String name, @NonNull String occupation) {
        this.name = name;
        this.occupation = occupation;
}
```


#### This is an example of model that will also be an entity in our database:
Every field that's stored in the database needs to be either public or have an accessor.
```
@Entity(tableName = "contact_table")
public class Contact {

    @PrimaryKey(autoGenerate = true)
    private int id;

    private String name;
    private String occupation;

    public Contact() {
    }

    //As id is a PK, it will be automatically generate, therefore, it doesn't need to go into the constructor.
    public Contact(@NonNull String name, @NonNull String occupation) {
        this.name = name;
        this.occupation = occupation;
    }

    public int getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getOccupation() {
        return occupation;
    }
}
```

### Data Access Objects (DAO) - Interface
The Data Access Objects is the API for the SQLiteDatabase. It is an interface that declares the CRUD methods for your database. The primary responsibility of the DAO is to map the SQL queries to functions that can be called. By default, all queeries must be exectured on a seperate thread (ExexcutorService).

#### @Dao
You use the @Dao annotation to let the compiler know that this interface will be a Data Access Object Interface.

```
@Dao
public interface ContactDao { }
```

#### @Insert(onConflict)
The @Insert(onConflict) annotation lets the compiler know that the method is as INSERT. There are a variety of OnConflictStrategies that you can use to handle conflicts.

```
@Insert (onConflict = OnConflictStrategy.IGNORE)
void insert(Contact contact);
````

#### @Query(SQL)
The @Query annotations lets the know that the method is a query, and more specifically, what kind of query the method is performing. You use SQL to declare your intentions for each of the query methods. 
```
@Query("DELETE FROM contact_table")
void deleteAll();

@Query("SELECT * FROM contact_table ORDER BY name ASC")
LiveData<List<Contact>> getAllContacts();
```

#### This is an example of a DAO interface:
```
@Dao
public interface ContactDao {

    //CRUD Operations
    @Insert (onConflict = OnConflictStrategy.IGNORE)
    void insert(Contact contact);

    @Query("DELETE FROM contact_table")
    void deleteAll();

    @Query("SELECT * FROM contact_table ORDER BY name ASC")
    LiveData<List<Contact>> getAllContacts();

}
```

### LiveData
LiveData follows the Observer pattern. It is an observerable data wrapper and it notifies it's observers when the data has changed. LiveData is lifecycle aware. It only cosniders an observer to be in an active state if its lifecycle is STARTED OR RESUMED. Inactive observers registed to watch LiveData objects aren't notified about changes. An instance of this object is generally created within your ViewModel class. If you want to use LiveData independently from Room, use MutableLiveData with the methods .setValue(T) and .postValue(T). 
```
LiveData<List<Contact>> getAllContacts();
```

### Room Database
Room is a database layer on top of an SQLiteDatabase. By default, queries are run on a background thread (to prevent the UI from being locked up). Your RoomDatabase class but be <strong>abstract and extend RoomDatabase</strong>. The documentation states that, generally, only one instance of a Room database is needed for the whole app. The RoomDatabase will be a singleton.

#### @Database(entities = {Contact.class}, version = 1, exportSchema = false)
This annotation lets the compiler now that this is our Room Database. The parameters declare the entities that belong in our database. The exportScehma parameter is for database migrations. When you modify the database schema, you'll need to update the version number and define a migration strategy.



#### getDatabase() 
Returns the RoomDatabase singleton (names the database "contact_database" if it's being acccesssed for the first time).

#### Executor Service
The executor service will be used to to run database operations on a background thread.


```
@Database(entities = {Contact.class}, version = 1, exportSchema = false)
public abstract class ContactRoomDatabase extends RoomDatabase {

    public abstract ContactDao contactDao();
    public static final int NUMBER_OF_THREADS = 4;

    private static volatile ContactRoomDatabase INSTANCE;
    private static final ExecutorService databaseWriteExecutor  = Executors.newFixedThreadPool(NUMBER_OF_THREADS);

    public static ContactRoomDatabase getDatabase(final Context context) {
        if (INSTANCE == null) {
            synchronized (ContactRoomDatabase.class) {
                if (INSTANCE == null) {
                    INSTANCE = Room.databaseBuilder(context.getApplicationContext(), ContactRoomDatabase.class, "contact_database").addCallback(sRoomDatabaseCallback).build();
                }
            }
        }
        return INSTANCE;
    }
```

### Repository
A repository makes the code easy to change, as it abstracts access to multiple data backends (network or local). Any queries should be performed on the background thread with the ExcutorService.

```
public class ContactRepository {

    //Properties
    private ContactDao contactDao;
    private LiveData<List<Contact>> allContacts;

    //Life Cycle
    public ContactRepository(Application application) {
        ContactRoomDatabase db = ContactRoomDatabase.getDatabase(application);
        contactDao = db.contactDao();

        allContacts = contactDao.getAllContacts();
    }

    public LiveData<List<Contact>> getAllData() { return allContacts; }
    
    public void insert(Contact contact) {
        ContactRoomDatabase.databaseWriteExecutor.execute(() -> {
            contactDao.insert(contact);
        });
    }
}
```

### ViewModel
The ViewModel is used in Android to manage configuration changes (among other things). When a device is rotated an activity is destroyed and recreated. Therefore, storing transient data in the activity class is not idea as it will not be retained during a reconfiguration. This is where the ViewModel comes in. The data is stored outside of the Activity. It is able to hold your app's UI data in a lifecycle-conscious way. In addition, seperating your UI data from Activity and Fragment allows your to follow the single responsibility principle.
- Activities and fragments are responsible for drawing data to the screen
- ViewModel stores and processes data needed for the UI

The repository keeps your ViewModel decoupled from any specific database. The ViewModel calls the method inset() and the repository decides where that data will be inserted. 


```
public class ContactViewModel extends AndroidViewModel {

    private final ContactRepository contactRepository;

    private final LiveData<List<Contact>> allContacts;

    public ContactViewModel(@NonNull Application application) {
        super(application);
        contactRepository = new ContactRepository(application);
        allContacts = contactRepository.getAllContacts();
    }

    LiveData<List<Contact>> getAllContacts() { return allContacts; }

    public void insert(Contact contact) { contactRepository.insert(contact); }
}
```

## Additional Considerations:

#### Database Inspector
Android Studio has a database inspector in the debug console. You can write your own SQL queries into the console. Amazing!
