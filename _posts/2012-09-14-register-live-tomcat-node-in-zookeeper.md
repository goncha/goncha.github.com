---
layout: post
title: "Register Online Tomcat Nodes in ZooKeeper"
tags: tomcat zookeeper
---

[Apache ZooKeeper](http://zookeeper.apache.org/) is a reliable distributed coordination. Here is an example to gather info of online tomcat nodes in it. ZooKeeper's API is difficult to use, please check [Netflix curator](http://netflix.github.com/curator).

	package tomcat.zookeeper;

	import java.io.IOException;
	import java.net.InetAddress;
	import java.net.NetworkInterface;
	import java.net.SocketException;
	import java.util.Enumeration;
	import java.util.Iterator;
	import java.util.LinkedList;
	import java.util.List;
	import java.util.concurrent.CountDownLatch;

	import org.apache.catalina.Lifecycle;
	import org.apache.catalina.LifecycleEvent;
	import org.apache.catalina.LifecycleListener;
	import org.apache.catalina.Service;
	import org.apache.catalina.connector.Connector;
	import org.apache.catalina.core.StandardServer;
	import org.apache.zookeeper.CreateMode;
	import org.apache.zookeeper.KeeperException;
	import org.apache.zookeeper.WatchedEvent;
	import org.apache.zookeeper.Watcher;
	import org.apache.zookeeper.ZooDefs;
	import org.apache.zookeeper.ZooKeeper;
	import org.apache.zookeeper.Watcher.Event.KeeperState;
	import org.apache.zookeeper.ZooKeeper.States;
	import org.apache.zookeeper.data.Stat;

	public class StandardServerListener implements LifecycleListener, Runnable {

		private static final String DEFAULT_SERVICE_NAME = "Catalina";
		
		private static final String ROOT_XNODE_PATH = "/tomcat";
		
		private static final byte[] EMPTY_DATA = new byte[0];
		
		private ZooKeeper zk;
		
		String nodeName;
		
		String connectStr;
		
		Thread monitorDaemon;

		@Override
		public void run() {
			while (true) {
				try {
					Thread.sleep(5000);
				} catch (InterruptedException e) {
					break;
				}
				
				States state = zk.getState();
				debug("State monitor: " + state.name());
				// when network link is up, state = CONNECTED
				// when network link is down, state = CONNECTING
				// when network link is up and session timeout, state = CLOSED
				// when network link is up and session alive, state = CONNECTED
				if (!state.isAlive()) {
					// if step here, network link is already up and session is timeout.
					// so we create a new client.
					try {
						zk = connectServer();
						createServerZnode();
					} catch (IOException e) {
						debug("Connecting to server failed: " + e.getMessage());
					}
					
				}
			}
		}

		@Override
		public void lifecycleEvent(LifecycleEvent event) {
			debugEvent(event);
			
			Lifecycle source = event.getLifecycle();
			if (source instanceof StandardServer) {
				if (Lifecycle.AFTER_START_EVENT.equals(event.getType())) {
					afterServerStart((StandardServer) source);
					// after connected to server, create a monitor thread
					monitorDaemon = new Thread(this);
					monitorDaemon.setDaemon(true);
					monitorDaemon.start();
				} else if (Lifecycle.AFTER_STOP_EVENT.equals(event.getType())) {
					afterServerStop((StandardServer) source);
				}
			}
		}

		static class ConnectWatcher implements Watcher {
			CountDownLatch lock;
			ConnectWatcher(CountDownLatch lock) {
				this.lock = lock;
			}
			
			@Override
			public void process(WatchedEvent event) {
				if (event.getType() == Watcher.Event.EventType.None) {
					KeeperState state = event.getState();
					if (state == KeeperState.SyncConnected) {
						lock.countDown();
					} else if (state == KeeperState.ConnectedReadOnly) {
						throw new RuntimeException("Connected read-only");
					}
				}
			}
		}
		
		
		void afterServerStart(StandardServer server) {
			nodeName = getServerKey(server);

			String connectStr = System.getProperty("zookeeper.connectStr");
			debug(connectStr);
			if (connectStr != null && connectStr.length() > 0) {
				this.connectStr = connectStr;
				try {
					zk = connectServer();
				} catch (IOException e) {
					debug("Connecting to server failed");
					throw new RuntimeException(e);
				}
				createServerZnode();
			} else {
				throw new IllegalArgumentException("Please add -Dzookeeper.connectStr=... to JAVA_OPTS");
			}
		}
		
		ZooKeeper connectServer() throws IOException {
			CountDownLatch lock = new CountDownLatch(1);
			debug("Connecting to " + this.connectStr);
			// if network link is down, the constructor will be blocking until link up
			ZooKeeper zk = new ZooKeeper(this.connectStr, 20000, new ConnectWatcher(lock));
			debug("Connected");
			
			// wait for session established asynchronously
			try {
				lock.await();
			} catch (InterruptedException e) {}
			
			return zk;
		}
		
		void afterServerStop(StandardServer server) {
			if (zk != null) {
				try {
					zk.close();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
			
			if (monitorDaemon != null) {
				monitorDaemon.interrupt();
			}
		}
		
		void createServerZnode() {
			try {
				ensureRootZnode();
				zk.create(ROOT_XNODE_PATH + "/" + nodeName, EMPTY_DATA, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
			} catch (Exception e) {
				throw new RuntimeException(e);
			}
		}

		void ensureRootZnode() throws KeeperException, InterruptedException {
			Stat stat = zk.exists(ROOT_XNODE_PATH, false);
			if (stat == null) {
				zk.create(ROOT_XNODE_PATH, EMPTY_DATA, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
			}
		}
		
		String getServerKey(StandardServer server) {
			String port = getListeningPort(server);
			List<String> inetAddresses = probeInetAddresses();
			
			StringBuilder sb = new StringBuilder();
			boolean first = true;
			Iterator<String> it = inetAddresses.iterator();
			while (it.hasNext()) {
				if (!first) sb.append(",");
				sb.append(it.next());
				sb.append(":").append(port);
				if (first == true) first = false;
			}
			return sb.toString();
		}
		
		List<String> probeInetAddresses() {
			List<String> foundAddrs = new LinkedList<String>();
			try {
				Enumeration<NetworkInterface> nics = NetworkInterface.getNetworkInterfaces();
			    while(nics.hasMoreElements()) {
			    	NetworkInterface nic = nics.nextElement();
			    	if (nic.isUp()) {
			    		Enumeration<InetAddress> addrs = nic.getInetAddresses();
			    		while (addrs.hasMoreElements()) {
			    			InetAddress addr = addrs.nextElement();
			    			if (!addr.isLoopbackAddress() && !addr.isAnyLocalAddress()) {
			    				foundAddrs.add(addr.getHostAddress());
			    			}
			    		}
			    	}
			    }
			} catch (SocketException e) {
				throw new RuntimeException(e);
			}
		    return foundAddrs;
		}
		
		/**
		 * Cannot get `address' of a Connector, so only return port number.
		 * @param server
		 * @return
		 */
		String getListeningPort(StandardServer server) {
			Service service = server.findService(DEFAULT_SERVICE_NAME);
			if (service == null) {
				throw new RuntimeException("No Service node with name " + DEFAULT_SERVICE_NAME + " in server.xml");
			}
			
			Connector[] connectors = service.findConnectors();
			if (connectors == null || connectors.length == 0) {
				throw new RuntimeException("No Connector defined in server.xml");			
			} else if (connectors.length == 1) {
				return Integer.toString(connectors[0].getPort());
			} else {
				throw new RuntimeException("Only one Connector is permitted to be defined in server.xml");
			}
		}

		void debugEvent(LifecycleEvent event) {
			StringBuilder sb = new StringBuilder();
			sb.append(event.getLifecycle().getClass().getCanonicalName());
			sb.append(", EVENT=");
			sb.append(event.getType());
			sb.append(", DATA=");
			sb.append(event.getData());
			debug(sb.toString());
		}
		
		void debug(String msg) {
			System.out.println("ZooKeeper>> " + msg);
		}
	}

How to inject this listener in Tomcat? Modify CATALINA_BASE/conf/server.xml, then drop your jar and ZooKeeper's jars in CATALINA_BASE/lib.

	<?xml version='1.0' encoding='utf-8'?>
	<Server port="8005" shutdown="SHUTDOWN">
	  ...
	  <!-- **** I'm here **** -->
	  <Listener className="tomcat.zookeeper.StandardServerListener"></Listener>
	  ...  
	  <Service name="Catalina">  
	    <Connector port="8080" protocol="HTTP/1.1" 
	               connectionTimeout="20000" 
	               redirectPort="8443" />
	     ...
	  </Service>
	</Server>

Start tomcat and use zkCli to check.