Example - Amazon

Charge -> Reserve -> Label -> Pick -> Email


Simple Apporach:
Client -> API Server -> (Payment Service, Inventory Service, Shipping Service, Warehouse service, Email Service) 

Problems that can occur -> 
1 - If API servier crashes
2 - CallBack Routing - Payment gateway gives response via a webhook. So if we have more than one API server and a load balancer in front of it we will not know which database to be called
3 - 
