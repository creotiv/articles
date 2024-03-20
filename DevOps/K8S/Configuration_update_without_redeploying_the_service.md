# Configuration update without redeploying the service in K8S

**Why this can be needed if we just can redeploy?**

1. Redeploy is taking much longer time then just config updates.

2. In HL projects redeploy may create additional load spikes, which we don't want to happen

3. Even with a graceful shutdown & `preStop hook` timeouts there is a chance that some of the requests can be broken.

4. It's much more complicated process, and thus it creates more risks. Because even good architected systems sometimes failing due to strange reasons.


**How we can achieve this:**

1. Use ConfigMap to store your configuration

2. Create config reload mechanism inside your application

3. Send signal to your services when configuration got changed.

3.1 You can use File Watcher inside the application (Simplest way)

3.2 You can send SIGHUP signal after you update configs (Manual way)

3.3 You can write a plugin that will watch for the config changes and will notify service with a signal(SIGHUP, http port/endpoint, etc)


