# sage
SAGE Version 6.1 SDK files
Link: https://smist08.wordpress.com/2013/01/01/the-sage-300-erp-java-api/

With version 6.0A of Sage 300 ERP we introduced a native Java API to all the Sage 300 Business Logic (Views). We did this in support of our SData implementation which we wrote in Java. This API allows Java programmers to access all the Sage 300 ERP business logic along the same lines as our .Net API and our COM API. This API isn’t built on top of either COM or .Net, it talks directly to the underlying C DLLs in System Manager. This then provides better performance, as well as allows us to compile this part of the system for Linux with no Microsoft dependencies. Internally we usually refer to this API as SAJava.
All the Sage 300 Business Logic objects have the same API, this makes it easier for us to produce these different APIs to facilitate interoperability with all sorts of external systems, allowing the programmers there to write code in a natural manner where any required interop layer is provided by us. The Java API uses a Java Native Interface (JNI) interop layer to talk to our Windows DLLs (or Linux shared objects). This is a one way communication where we only use this to call the DLLs, we never have the DLLs calling our Java code (this direction is often dangerous and leads to the problems often encountered with JNI). Our JNI code handles all the data conversions between Java and C as well as provides exception handling to trap and handle exceptions that can happen in C code (like bad pointers).
I’ve blogged about this API a bit indirectly in the past when talking about how to write server side code for our SData service, for instance here, here and here. Generally to add custom programming to SData feeds you write Java classes that inherit from our standard SData classes to provide this. When you interact with the Views in this environment you use this Java API, but all the libraries are already included and all the details of signing on are handled for you. The framework starts you off at a point where you can directly open and call Views. In this posting we’ll back up a bit to cover full usage, unlike the case where the SData programming framework does a lot of the work for you. So that you can use this API directly in isolation without requiring any other framework.
Getting Started
First to use the Java API, you need to include its jar file into your project. This file is located in the Tomcat\lib folder. This changed a bit between version 6.0A and then the 2012 version. For 6.0A the folder is: C:\Program Files (x86)\Common Files\Sage\Sage ERP Accpac\Tomcat\lib and the file is SystemManager.jar. For the 2012 version the folder is: C:\Program Files (x86)\Common Files\Sage\Sage 300 ERP\Tomcat\lib and the file is com.sage.accpac.sdk.accpac.sajava-6.1.jar. Then you need to import the classes into any source file that uses them via:
import com.sage.accpac.sm.*;

Once you have these things included in your Java project you can start creating objects and calling methods. However due to security you first must sign-on to a session and then create all other objects from this session.
The documentation is in the form of JavaDoc and is located on the DPP Wiki. The 2012 version is here: http://dppwiki.sage300erp.com//javadocs/v6.1/SystemManager/. You can find all the classes, methods and properties here. To access this, you must be part of the Sage 300 ERP Developer Program. A key benefit to joining this program is access to this wiki which contains all the developer documentation that we produce.
Signing On
First you must create a session with some code like:
Session session;
session = new Session(new ProgramSet(), new SharedDataSet(), "ADMIN",
     "ADMIN", "SAMINC", new Date());

This will sign-on your session to the company SAMINC using today’s date as the session date. The ProgramSet and SharedDataSet are used when we deploy in a hosted configuration and run multi-tenant. In this case they must be setup correctly by the system to configure which tenant this session is for. In most normal on-premise applications the indicated calls are fine to give the one default tenant that exists.
Then you must create a program from the session:
Program program;
program = new Program(session, "XZ", "XZ0001", "61A");

If you read my last blog post, this might appear a bit backwards to the COM API where this looks like the session.Init call that comes first. This is true, but the information is required regardless.
Using Views
Now that you have a program you can start opening and using Views. As an example, let’s look at a method that enters A/R Invoices. Like many things I started with macro recording to get the right Views and some syntax. Macro recording produces VBA code, but it isn’t hard to convert this to Java quickly. Anyone familiar with Sage 300 ERP macro recording will recognize the style and variable names in the following method. This method assumes there are class variables for the program and session that were created as indicated above. The key point of the following example is to show how to open Views, compose Views and then use the Views. For more general information on Sage 300 ERP’s Views have a look at this and this.
    public String enterARInvoices()

    {

        int iEntry;
        int iDetail;
        int numEntries = 20;
        int numDetails = 5;
        String sBatchNum;

        View ARINVOICE1batch = new View(program, "AR0031");
        View ARINVOICE1header = new View(program, "AR0032");
        View ARINVOICE1detail1 = new View(program, "AR0033");
        View ARINVOICE1detail2 = new View(program, "AR0034");
        View ARINVOICE1detail3 = new View(program, "AR0402");
        View ARINVOICE1detail4 = new View(program, "AR0401");
        View ARCUSTOMER1header = new View(program, "AR0024");

        ARINVOICE1batch.compose ( ARINVOICE1header );
        ARINVOICE1header.compose (ARINVOICE1batch, ARINVOICE1detail1, ARINVOICE1detail2, ARINVOICE1detail3, null);
        ARINVOICE1detail1.compose (ARINVOICE1header, ARINVOICE1batch, ARINVOICE1detail4);
        ARINVOICE1detail2.compose (ARINVOICE1header);
        ARINVOICE1detail3.compose (ARINVOICE1header);
        ARINVOICE1detail4.compose (ARINVOICE1detail1);

        // Create the batch

        ARINVOICE1batch.recordGenerate(RecordGenerateMode.Insert);
        ARINVOICE1batch.set("PROCESSCMD","1");      // Process Command

        ARINVOICE1batch.process();
        ARINVOICE1batch.read(false);

        sBatchNum = ARINVOICE1batch.get("CNTBTCH").toString();

        // Loop through creating the entries

        for ( iEntry = 0; iEntry < numEntries; iEntry++ )
        {
            try
            {
                ARINVOICE1detail1.cancel();
                ARINVOICE1detail2.cancel();
                ARINVOICE1header.recordGenerate(RecordGenerateMode.DelayKey);
                ARINVOICE1detail1.recordClear();
                ARINVOICE1detail2.recordClear();

                ARINVOICE1header.set("PROCESSCMD","4");

                ARINVOICE1header.process();

                if ( false == ARCUSTOMER1header.goNext() )
                {
                    ARCUSTOMER1header.goTop();
                }

                ARINVOICE1header.set("IDCUST", "1200");

                for ( iDetail = 0; iDetail < numDetails; iDetail++ )
                {
                    ARINVOICE1detail1.recordClear();
                    ARINVOICE1detail1.recordGenerate (RecordGenerateMode.NoInsert);
                    ARINVOICE1detail1.process();

                    ARINVOICE1detail1.set("IDITEM", "CA-78" );                     // Item Number

                    ARINVOICE1detail1.insert();

                }

                ARINVOICE1header.insert();
            }
            catch( Exception e )
            {
                int count = program.getErrors().getCount();
                if ( 0 == count )
                {
                    e.printStackTrace();                   
                }
                for ( int i = 0; i < count; i++ )
                {
                    System.out.println(program.getErrors().get(i).getMessage());
                }
            }
        }
        ARINVOICE1batch.dispose();
        ARINVOICE1header.dispose();
        ARINVOICE1detail1.dispose();
        ARINVOICE1detail2.dispose();
        ARINVOICE1detail3.dispose();
        ARINVOICE1detail4.dispose();
        ARCUSTOMER1header.dispose();

        return( sBatchNum );
    }
Notice that you can explicitly close things by calling the dispose method. This is usually preferred to waiting for the Java garbage collector to reclaim things, it tends to keep down resource usage if you are opening and closing things a lot.
Errors
If a call fails, there are a couple of cases. If it’s a simple expected thing like reaching the end of records when fetching through them then the routine will return a simple return code that you can easily handle in your code. If something worse happens then the routine will throw an exception. As in other Sage 300 ERP APIs, there is an error stack which will contain possibly a number of error messages explaining what went wrong. In the catch expression above we first check if there are any errors on the error stack, if not then we print the stack trace to allow debugging of what went wrong. Otherwise we loop through the Sage 300 errors and print them for diagnostic purposes. When programming Sage 300 ERP, always make sure you have an error handler as it can give you very good information when debugging your program.
Summary
The Sage 300 ERP Java API gives yet another tool for integrators to integrate to Sage 300 ERP from external systems. It is ideal for Java programmers who would like to write their integration entirely in Java. This is often a benefit when the SDK for the external system is itself written around the Java programming language.
