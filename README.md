#spring-boot-akka-java
## A web application involving Spring Boot and Akka using Java 8

----
## Preface

> As a beginner, I found integrating Akka with Spring's dependency injection a bit complex, especially if I am using Java. This application is a step forward to do the same but in a much simplified way (rather than using Spring Extension and ApplicationConfiguration codes). The application has been built using Gradle. To run it one has go inside "spring-boot-akka-java" folder and type "gradle bootRun". Then open the browser and type

    http://localhost:8080/task/distribute

>This will enter different Task objects into your database

The application contains a domain/model class named Task in the package "com.example.domain" mapped to a databse table of the same name.

    @Entity
    @Table(name = "task")
    public class Task {
        @Id
        @GeneratedValue(strategy = GenerationType.AUTO)
        private Long id;
        @Version
        private Long version;
        private String name;
        private String description;
        ...
    }
    
The data access is done using the TaskRepository interface residing inside package "com.example.repository" package. This interface simply extends the "org.springframework.data.repository.CrudRepository" interface. 

    @Repository
    public interface TaskRepository extends CrudRepository<Task, Long> {
    }

The TaskServiceImpl class inside package "com.example.service", is responsible for generating new "Task" objects and routing them to different "TaskActor"s routed through the a "TaskRouter". The TaskRepository class is injected through a constructor.

    @Service
    public class TaskServiceImpl implements ITaskService {
        TaskRepository taskRepository;
        @Autowired
        public TaskServiceImpl(TaskRepository taskRepository){
            this.taskRepository = taskRepository;
        }
        @Override
        @Transactional
        public void distributeTasks() {
            final ActorSystem system = ActorSystem.create("TaskSystem",
                    ConfigFactory.load().getConfig("TaskSystem"));
            ActorRef taskRouter = system.actorOf(
                    TaskActor.props(taskRepository).withRouter(new FromConfig()),
                    "TaskRouter");
            for (int i = 1; i < 12; i++) {
                Task task = new Task();
                task.setName("Task " + i);
                task.setDescription("Description of task "+i);
                taskRouter.tell(task, null);
            }
        }
    }

The ActorSystem is created from a configuration file "application.conf" residing in the "resources" folder.

    TaskSystem{
	akka{
		actor{
			deployment {
				  /TaskRouter {
				    router = smallest-mailbox-pool
				    nr-of-instances = 4
				  }
	  		}
      }
	}
}

The router responsible for routing tasks to the actors can now be created as provided below.

    ActorRef taskRouter = system.actorOf(
                TaskActor.props(taskRepository).withRouter(new FromConfig()),
                "TaskRouter")
                
I wanted to have access to the "TaskRepository" (created earlier) from the Actors. This "TaskRepository" was passed as a constructor arguement while creating the actor. This is where I have some doubts about its side effects, because, I did not find anyone creating Actors like this before. Nevertheless, it worked as of now.

    @Scope("prototype")
    public class TaskActor extends UntypedActor{

    private TaskRepository taskRepository;

    public static Props props(final TaskRepository taskRepository) {
        return Props.create(new Creator<TaskActor>() {
            private static final long serialVersionUID = 1L;
            @Override
            public TaskActor create() throws Exception {
                return new TaskActor(taskRepository);
            }
        });
    }

    public TaskActor(TaskRepository taskRepository){
        this.taskRepository = taskRepository;
    }


    @Override
    @Transactional
    public void onReceive(Object message) throws Exception {

        Task task = (Task) message;
        System.out.println(task.getName());
        taskRepository.save(task);

    }

    }

