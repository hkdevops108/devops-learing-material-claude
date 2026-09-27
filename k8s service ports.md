# Why `port`, `targetPort`, and `nodePort` Exist

Don't memorize `80`, `8080`, `30000` etc. First understand **why Kubernetes needs these ports**.

Imagine your application is running in a Pod:

```text
Pod
┌──────────────────────────┐
│                          │
│   My application         │
│   listening on :5000     │
│                          │
└──────────────────────────┘
```

The application already works on `5000`.

So you may ask:

> "Why do I need `port`, `targetPort`, and `nodePort` at all?"

Because **the Pod's application and the Kubernetes network are two different things.**

---

## 1. What if we don't have a Service?

Suppose you have:

```text
Pod
Application :5000
```

You could potentially access the Pod directly using its Pod IP:

```text
10.244.1.5:5000
```

But there is a problem.

Pods are **temporary**.

Today:

```text
Pod A
IP = 10.244.1.5
```

Tomorrow Kubernetes deletes/recreates it:

```text
Pod B
IP = 10.244.2.8
```

Now everyone who was using:

```text
10.244.1.5:5000
```

has a problem.

That's one major reason we use a **Service**.

```text
             Service
                |
       ┌────────┼────────┐
       ↓        ↓        ↓
     Pod A    Pod B    Pod C
```

The Service gives us a **stable way to reach the Pods**.

---

## 2. Why `targetPort`?

Suppose your application listens on:

```text
5000
```

Kubernetes needs to know:

> "Which port should I send the traffic to inside the Pod?"

That's what:

```yaml
targetPort: 5000
```

tells Kubernetes.

```text
Service
   |
   | "Send this traffic to port 5000"
   ↓
Pod
   |
   ↓
Application :5000
```

### What if `targetPort` isn't specified?

Kubernetes has a default behavior: if `targetPort` is omitted, it defaults to the value of `port`.

So if you write:

```yaml
ports:
  - port: 80
```

Kubernetes effectively targets:

```text
Pod :80
```

If your application is actually listening on `5000`, that won't work unless the application is also listening on 80.

So you **don't always need to write `targetPort`**, but you need the resulting target to match your application.

---

## 3. Why `port`?

Now imagine another application inside Kubernetes wants to access your application.

You don't want it to worry about:

```text
Which Pod?
Which Pod IP?
Which application port?
```

Instead, it can simply use the Service:

```text
my-app-service:80
```

The Service receives the request on:

```text
port: 80
```

and sends it to:

```text
targetPort: 5000
```

So:

```text
Other Pod
    |
    | my-app-service:80
    ↓
 Service :80
    |
    | forwards
    ↓
 Pod :5000
    |
    ↓
Application
```

### What if `port` isn't there?

For a normal Service, **`port` is required** in the Service port definition.

Why?

Because Kubernetes needs to know:

> "On which port should this Service accept traffic?"

---

## 4. Why `nodePort`?

Now let's say someone **outside the Kubernetes cluster** wants to access your application.

For example:

```text
Your laptop
     |
     ↓
Kubernetes cluster
     |
     ↓
Pod
```

A normal `ClusterIP` Service is primarily for **inside-cluster communication**.

If you want to expose the Service through a Kubernetes Node using `NodePort`, you can write:

```yaml
type: NodePort
```

and:

```yaml
nodePort: 30050
```

Then:

```text
Outside
   |
   | NodeIP:30050
   ↓
Node
   |
   ↓
Service
   |
   ↓
Pod
```

### What if `nodePort` isn't there?

That's completely okay **if you don't need NodePort access**.

For example:

```yaml
type: NodePort

ports:
  - port: 80
    targetPort: 5000
```

You don't specify `nodePort`.

Kubernetes can automatically allocate a NodePort for you.

So **you don't always have to manually write `nodePort`**.

---

## The BIG picture

This is the part I want you to remember:

```text
                   KUBERNETES
                       
Outside
   |
   | nodePort
   ↓
 Node
   |
   ↓
Service
   |
   | port
   ↓
Service networking
   |
   | targetPort
   ↓
 Pod
   |
   ↓
Application
```

But don't think of `port` as physically "going into" the Pod. Think of it as the **Service's listening/entry port**, while `targetPort` tells the Service where to forward traffic.

---

## Do I always need all 3?

**No!** This is very important.

### Internal application

If you only need communication inside the cluster:

```yaml
type: ClusterIP

ports:
  - port: 80
    targetPort: 5000
```

You don't need `nodePort`.

```text
Pod A
  |
  | my-service:80
  ↓
Service
  |
  ↓
Pod B:5000
```

---

### External access using NodePort

Then:

```yaml
type: NodePort

ports:
  - port: 80
    targetPort: 5000
    nodePort: 30050
```

Now:

```text
Outside
   |
   | :30050
   ↓
Node
   ↓
Service :80
   ↓
Pod :5000
```

---

## 🧠 The easiest way to remember

Don't memorize numbers.

Remember **three questions**:

### Question 1

**"Where is my application listening?"**

➡️ `targetPort`

```text
Application → 5000
targetPort → 5000
```

### Question 2

**"Which port should my Service provide inside the cluster?"**

➡️ `port`

```text
Service → 80
port → 80
```

### Question 3

**"Do I want to expose this through a Node to outside users?"**

➡️ `nodePort`

```text
Outside → 30050
nodePort → 30050
```

So the mental picture is:

```text
             SERVICE
          ┌─────────────┐
          │             │
Inside →  │ port : 80   │
          │             │
          └──────┬──────┘
                 |
                 | targetPort : 5000
                 ↓
             Application
                :5000

Outside → NodePort :30050
```

**Most importantly:** `targetPort` is about **your application**, `port` is about **your Service**, and `nodePort` is about **external access through a Node**. Once you understand those three purposes, the numbers become much easier.
