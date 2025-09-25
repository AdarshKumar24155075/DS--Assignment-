# DS--Assignment-
Done by Sayonika (24255124)
    Adarsh kumar (24155075)
            
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <math.h>

#define MAX_CUSTOMERS 1000
#define MAX_TELLERS 10
#define IDLE_MIN 1
#define IDLE_MAX 150
#define TELLER_IDLE_MAX 600

// Event Types
typedef enum { ARRIVAL, SERVICE_COMPLETE, TELLER_IDLE } EventType;

// Forward declaration of Event
struct Event;

// Function pointer type for event actions
typedef void (*ActionFn)(struct Event *);

// Customer structure
typedef struct Customer {
    int id;
    float arrivalTime;
    float serviceStartTime;
    float serviceEndTime;
    struct Customer *next;
} Customer;

// Teller structure
typedef struct Teller {
    int id;
    float idleTime;
    float totalServiceTime;
    float totalIdleTime;
    Customer *queueHead;
    Customer *queueTail;
    int queueLength;
} Teller;

// Event structure
typedef struct Event {
    EventType type;
    float time;
    int tellerId;
    Customer *customer;
    ActionFn action;
    struct Event *next;
} Event;

// Event Queue (time ordered linked list)
typedef struct {
    Event *head;
} EventQueue;

EventQueue eventQueue = {NULL};
Customer customers[MAX_CUSTOMERS];
Teller tellers[MAX_TELLERS];

// Simulation parameters
int numCustomers, numTellers;
float simulationTime, avgServiceTime;
float totalWaitTime = 0;
float maxWaitTime = 0;
int servedCustomers = 0;

// ---------- Utility Functions ----------

// Generate uniform random float
float randFloat(float max) {
    return max * ((float) rand() / RAND_MAX);
} 

// Insert event into time-ordered queue
void addEvent(Event *newEvent) {
    if (!eventQueue.head || (*newEvent).time < (*eventQueue.head).time) {
        (*newEvent).next = eventQueue.head;
        eventQueue.head = newEvent;
    } else {
        Event *cur = eventQueue.head;
        while (cur->next && (cur->next)->time <= (*newEvent).time) { //(*ptr).data == ptr -> data
            cur = (*cur).next;
        }
        (*newEvent).next = (*cur).next;
        (*cur).next = newEvent;
    }
}

// Pop first event from queue
Event* popEvent() {
    if (!eventQueue.head) return NULL;
    Event *e = eventQueue.head;
    eventQueue.head = (*eventQueue.head).next;
    return e;
}

// Create new event
Event* createEvent(EventType type, float time, int tellerId, Customer *customer, ActionFn action) {
    struct Event e = (struct Event) malloc(sizeof(struct Event));
    (*e).type = type;
    (*e).time = time;
    (*e).tellerId = tellerId;
    (*e).customer = customer;
    (*e).action = action;
    (*e).next = NULL;
    return e;
}

// Add customer to teller queue
void enqueueCustomer(Teller *teller, Customer *c) {
    (*c).next = NULL;
    if (!(*teller).queueHead) {
        (*teller).queueHead = (*teller).queueTail = c;
    } else {
        (*(*teller).queueTail).next = c;
        (*teller).queueTail = c;
    }
    (*teller).queueLength++;
}

// Remove customer from teller queue
Customer* dequeueCustomer(Teller *teller) {
    if (!(*teller).queueHead) return NULL;
    Customer *c = (*teller).queueHead;
    (*teller).queueHead = (*c).next;
    if (!(*teller).queueHead) (*teller).queueTail = NULL;
    (*teller).queueLength--;
    return c;
}

// ---------- Event Actions ----------

// Arrival event
void arrivalAction(Event *e) {
    Customer *c = (*e).customer;
    int minQueueLen = tellers[0].queueLength;
    int chosenTeller = 0;

    // Pick shortest teller queue
    for (int i = 1; i < numTellers; i++) {
        if (tellers[i].queueLength < minQueueLen) {
            minQueueLen = tellers[i].queueLength;
            chosenTeller = i;
        }
    }

    enqueueCustomer(&tellers[chosenTeller], c);

    // If teller is idle, immediately serve
    if (tellers[chosenTeller].queueLength == 1) {
        float serviceTime = 2 * avgServiceTime * ((float)rand() / RAND_MAX);
        (*c).serviceStartTime = (*e).time;
        (*c).serviceEndTime = (*e).time + serviceTime;
        tellers[chosenTeller].totalServiceTime += serviceTime;

        addEvent(createEvent(SERVICE_COMPLETE, (*c).serviceEndTime, chosenTeller, c, arrivalAction));
    }
}

// Service completion event
void serviceCompleteAction(Event *e) {
    Customer *c = (*e).customer;
    Teller *t = &tellers[(*e).tellerId];

    float waitTime = (*c).serviceEndTime - (*c).arrivalTime;
    totalWaitTime += waitTime;
    if (waitTime > maxWaitTime) maxWaitTime = waitTime;
    servedCustomers++;

    // Next customer in line
    Customer *next = dequeueCustomer(t);
    if (next) {
        float serviceTime = 2 * avgServiceTime * ((float)rand() / RAND_MAX);
        (*next).serviceStartTime = (*e).time;
        (*next).serviceEndTime = (*e).time + serviceTime;
        (*t).totalServiceTime += serviceTime;
        addEvent(createEvent(SERVICE_COMPLETE, (*next).serviceEndTime, (*e).tellerId, next, serviceCompleteAction));
    } else {
        // Teller idle
        float idleTime = IDLE_MIN + randFloat(IDLE_MAX);
        (*t).totalIdleTime += idleTime;
        addEvent(createEvent(TELLER_IDLE, (*e).time + idleTime, (*e).tellerId, NULL, serviceCompleteAction));
    }
}

// Teller idle event
void tellerIdleAction(Event *e) {
    Teller *t = &tellers[(*e).tellerId];
    Customer *next = dequeueCustomer(t);

    if (next) {
        float serviceTime = 2 * avgServiceTime * ((float)rand() / RAND_MAX);
        (*next).serviceStartTime = (*e).time;
        (*next).serviceEndTime = (*e).time + serviceTime;
        (*t).totalServiceTime += serviceTime;
        addEvent(createEvent(SERVICE_COMPLETE, (*next).serviceEndTime, (*e).tellerId, next, serviceCompleteAction));
    }
}

// ---------- Simulation ----------

void runSimulation() {
    Event *e;
    while ((e = popEvent())) {
        (*e).action(e);
        free(e);
    }
}

// ---------- Main ----------

int main(int argc, char *argv[]) {
    if (argc != 5) {
        printf("Usage: ./qSim #customers #tellers simulationTime averageServiceTime\n");
        return 1;
    }

    srand(time(NULL));

    numCustomers = atoi(argv[1]);
    numTellers = atoi(argv[2]);
    simulationTime = atof(argv[3]);
    avgServiceTime = atof(argv[4]);

    // Initialize tellers
    for (int i = 0; i < numTellers; i++) {
        tellers[i].id = i;
        tellers[i].idleTime = 0;
        tellers[i].totalServiceTime = 0;
        tellers[i].totalIdleTime = 0;
        tellers[i].queueHead = tellers[i].queueTail = NULL;
        tellers[i].queueLength = 0;
    }

    // Create customers with random arrival times
    for (int i = 0; i < numCustomers; i++) {
        customers[i].id = i;
        customers[i].arrivalTime = simulationTime * ((float) rand() / RAND_MAX);
        customers[i].next = NULL;
        addEvent(createEvent(ARRIVAL, customers[i].arrivalTime, -1, &customers[i], arrivalAction));
    }

    runSimulation();

    // Results
    printf("\nSimulation Results:\n");
    printf("Customers Served: \n");
    scanf("%d",&servedCustomers);
    printf("Total wait time: \n");
    scanf("%d",&totalWaitTime);
    printf("Maximum wait time: \n");
    scanf("%f",&maxWaitTime);
    float avg = (totalWaitTime / servedCustomers);
    printf("Average Time in Bank: %f minutes\n",avg);
    printf("Max Wait Time: %.2f minutes\n", maxWaitTime);
    printf("No of Tellers:\n");
    scanf("%d",&numTellers);
    for (int i = 0; i < numTellers; i++) {
        printf("Teller %d: Service Time = %.2f, Idle Time = %.2f\n",
               i, tellers[i].totalServiceTime, tellers[i].totalIdleTime);
    }

   return 0;
}            
            

            
