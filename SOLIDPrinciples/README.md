### OOPS Crash Course

## SOLID Design Principles
- Single Responsibility Principle: A class/method should do one thing e.g. like driving and texting is not good same goes with a method or a class that does two things simultaneously
- Open and Close Principle: The changes should extend the functionality of the existing code not modify it e.g if you are renovating yoir house probably its a good idea to add an extra room, extend patio but never good to change the basement and that risks breaking the entire house
- Liskov Substitution Principle: The derived class should have the same behavior as the base class. The derived class must honor the contracts of the interface. 

  Example of violation of liskov substitution principle: In mathematics, a Square is a Rectangle. Indeed it is a specialization of a rectangle. The "is a" makes you want to model this with inheritance. However if in code you made Square derive from Rectangle, then a Square should be usable anywhere you expect a Rectangle. This makes for some strange behavior.

  Imagine you had SetWidth and SetHeight methods on your Rectangle base class; this seems perfectly logical. However if your Rectangle reference pointed to a Square, then SetWidth and SetHeight doesn't make sense because setting one would change the other to match it. In this case Square fails the Liskov Substitution Test with Rectangle and the abstraction of having Square inherit from Rectangle is a bad one. Here is why:
  
  Example from [How to verify the Liskov substitution principle in an inheritance hierarchy?](https://softwareengineering.stackexchange.com/questions/170189/how-to-verify-the-liskov-substitution-principle-in-an-inheritance-hierarchy/170191#170191)
  
  
  For example originally :
  
  ```
  public class Square : Rectangle
  {
      public Square(double width) : base(width, width)
      {
      }

      public override double Width
      {
          set
          {
              base.Width = value;
              base.Height = value;
          }
          get
          {
              return base.Width;
          }
      }

      public override double Height
      {
          set
          {
              base.Width = value;
              base.Height = value;
          }
          get
          {
              return base.Height;
          }
      }
  }
  ```
  
    Seems fair enough, right? I've created a specialist kind of Rectangle called Square, which maintains that Width must equal Height at all times. A square is a rectangle, so it fits with OO principles, doesn't it?

    But wait, what if someone now writes this method (notice how this will reduce the width and height twice):
    
    ```
    public void Enlarge(Rectangle rect, double factor)
    {
        rect.Width *= factor;
        rect.Height *= factor;
    }
    ```

Liskov Substitution Principle requires that
  - Preconditions cannot be strengthened in a subtype.
  - Postconditions cannot be weakened in a subtype.
  - Invariants of the supertype must be preserved in a subtype.
  - History constraint (the "history rule"). Objects are regarded as being modifiable only through their methods (encapsulation). Since subtypes may introduce methods that are not present in the supertype, the introduction of these methods may allow state changes in the subtype that are not permissible in the supertype. The history constraint prohibits this.

  Violation Example: This example breaks the first principle
  ```
    public class Task
    {
        public Status Status { get; set; }

        public virtual void Close()
        {
            Status = Status.Closed;
        }
    }

    public class ProjectTask : Task
    {
        public override void Close()
        {
            if (Status == Status.Started) 
                throw new Exception("Cannot close a started Project Task");

            base.Close();
        }
    }
  ```

  Fix:
  ```
  public class Task {
    public Status Status { get; set; }
    public virtual bool CanClose() {
        return true;
    }
    public virtual void Close() {
        Status = Status.Closed;
    }
  }

  public class ProjectTask : Task {
    public override bool CanClose() {
        return Status != Status.Started;
    }
    public override void Close() {
        if (Status == Status.Started) 
            throw new Exception("Cannot close a started Project Task");
        base.Close();
    }
  }
  ```

  This principle encourages the usage of interfaces as much as possible. Here is why?
    - It requires very less extra effort
    - Never hurts
    - Comes handy quite often e.g. unit testing

- Interface Segregation Principle: The client should not be forced to depend on methods it doesn't use and that is why C# has IEnumerable, ICollection and IList interfaces
- Dependency Injection

## Best Practices

- DRY (Do not repeat yourself)
- The boy scount priciple: leave something cleaner than you found it
- Separation of concerns: macro level of single responsibility principle e.g. using DAL to connect with the database

## Design Patters
Remember BSC
- Behavior:
  - Strategy
  - Commands
- Structural:
  - Adaptor
  - Facade
- Creational:
  - Factory
  - Singleton    
