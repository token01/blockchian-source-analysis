server是p2p的最主要的部分。集合了所有之前的组件。

首先看看Server的结构
```
// Server manages all peer connections.
type Server struct {
	// Config fields may not be modified while the server is running.
	Config

	// Hooks for testing. These are useful because we can inhibit
	// the whole protocol stack.
	newTransport func(net.Conn, *ecdsa.PublicKey) transport
	newPeerHook  func(*Peer)
	listenFunc   func(network, addr string) (net.Listener, error)

	lock    sync.Mutex // protects running
	running bool

	listener     net.Listener
	ourHandshake *protoHandshake
	loopWG       sync.WaitGroup // loop, listenLoop
	peerFeed     event.Feed
	log          log.Logger

	nodedb    *enode.DB
	localnode *enode.LocalNode
	ntab      *discover.UDPv4
	DiscV5    *discover.UDPv5
	discmix   *enode.FairMix
	dialsched *dialScheduler

	// Channels into the run loop.
	quit                    chan struct{}
	addtrusted              chan *enode.Node
	removetrusted           chan *enode.Node
	peerOp                  chan peerOpFunc
	peerOpDone              chan struct{}
	delpeer                 chan peerDrop
	checkpointPostHandshake chan *conn
	checkpointAddPeer       chan *conn

	// State of run loop and listenLoop.
	inboundHistory expHeap
}

type peerOpFunc func(map[enode.ID]*Peer)

type peerDrop struct {
	*Peer
	err       error
	requested bool // true if signaled by the peer
}

type connFlag int32

const (
	dynDialedConn connFlag = 1 << iota
	staticDialedConn
	inboundConn
	trustedConn
)

// conn wraps a network connection with information gathered
// during the two handshakes.
type conn struct {
	fd net.Conn
	transport
	node  *enode.Node
	flags connFlag
	cont  chan error // The run loop uses cont to signal errors to SetupConn.
	caps  []Cap      // valid after the protocol handshake
	name  string     // valid after the protocol handshake
}

type transport interface {
	// The two handshakes.
	doEncHandshake(prv *ecdsa.PrivateKey) (*ecdsa.PublicKey, error)
	doProtoHandshake(our *protoHandshake) (*protoHandshake, error)
	// The MsgReadWriter can only be used after the encryption
	// handshake has completed. The code uses conn.id to track this
	// by setting it to a non-nil value after the encryption handshake.
	MsgReadWriter
	// transports must provide Close because we use MsgPipe in some of
	// the tests. Closing the actual network connection doesn't do
	// anything in those tests because MsgPipe doesn't use it.
	close(err error)
}
```

并不存在一个newServer的方法。 初始化的工作放在Start()方法中。

```
// Start starts running the server.
// Servers can not be re-used after stopping.
func (srv *Server) Start() (err error) {
	srv.lock.Lock()
	defer srv.lock.Unlock()
	if srv.running { //避免多次启动。 srv.lock为了避免多线程重复启动
		return errors.New("server already running")
	}
	srv.running = true
	srv.log = srv.Config.Logger
	if srv.log == nil { //判断是否为根节点启动
		srv.log = log.Root()
	}
	if srv.clock == nil {
		srv.clock = mclock.System{}
	}
	if srv.NoDial && srv.ListenAddr == "" {
		srv.log.Warn("P2P server will be useless, neither dialing nor listening")
	}

	// static fields
	if srv.PrivateKey == nil {
		return errors.New("Server.PrivateKey must be set to a non-nil key")
	}
	if srv.newTransport == nil { //这里注意的是Transport使用了newRLPX 使用了rlpx.go中的网络协议。
		srv.newTransport = newRLPX
	}
	if srv.listenFunc == nil {
		srv.listenFunc = net.Listen
	}
	srv.quit = make(chan struct{})
	srv.delpeer = make(chan peerDrop)
	srv.checkpointPostHandshake = make(chan *conn)
	srv.checkpointAddPeer = make(chan *conn)
	srv.addtrusted = make(chan *enode.Node)
	srv.removetrusted = make(chan *enode.Node)
	srv.peerOp = make(chan peerOpFunc)
	srv.peerOpDone = make(chan struct{})

	if err := srv.setupLocalNode(); err != nil {
		return err
	}
	if srv.ListenAddr != "" {
		// 开启监听tcp端口
		if err := srv.setupListening(); err != nil {
			return err
		}
	}
	if err := srv.setupDiscovery(); err != nil {
		return err
	}
	srv.setupDialScheduler()

	srv.loopWG.Add(1)
	//启动horoutine 来处理程序
	go srv.run()
	return nil
}
```
启动监听。 可以看到是TCP协议。 这里的监听端口和UDP的端口是一样的。 默认都是30303
	
	func (srv *Server) startListening() error {
		// Launch the TCP listener.
		listener, err := net.Listen("tcp", srv.ListenAddr)
		if err != nil {
			return err
		}
		laddr := listener.Addr().(*net.TCPAddr)
		srv.ListenAddr = laddr.String()
		srv.listener = listener
		srv.loopWG.Add(1)
		go srv.listenLoop()
		// Map the TCP listening port if NAT is configured.
		if !laddr.IP.IsLoopback() && srv.NAT != nil {
			srv.loopWG.Add(1)
			go func() {
				nat.Map(srv.NAT, srv.quit, "tcp", laddr.Port, laddr.Port, "ethereum p2p")
				srv.loopWG.Done()
			}()
		}
		return nil
	}

listenLoop()。 这是一个死循环的goroutine。 会监听端口并接收外部的请求。
```

// listenLoop runs in its own goroutine and accepts
// inbound connections.
func (srv *Server) listenLoop() {
	srv.log.Debug("TCP listener up", "addr", srv.listener.Addr())

	// The slots channel limits accepts of new connections.
	tokens := defaultMaxPendingPeers
	if srv.MaxPendingPeers > 0 {
		tokens = srv.MaxPendingPeers
	}
	//创建maxAcceptConns个槽位。 我们只同时处理这么多连接。 多了也不要。
	slots := make(chan struct{}, tokens)
	//把槽位填满
	for i := 0; i < tokens; i++ {
		slots <- struct{}{}
	}

	// Wait for slots to be returned on exit. This ensures all connection goroutines
	// are down before listenLoop returns.
	defer srv.loopWG.Done()
	defer func() {
		for i := 0; i < cap(slots); i++ {
			<-slots
		}
	}()

	for {
		// Wait for a free slot before accepting.
		<-slots

		var (
			fd      net.Conn
			err     error
			lastLog time.Time
		)
		for {
			fd, err = srv.listener.Accept()
			if netutil.IsTemporaryError(err) {
				if time.Since(lastLog) > 1*time.Second {
					srv.log.Debug("Temporary read error", "err", err)
					lastLog = time.Now()
				}
				time.Sleep(time.Millisecond * 200)
				continue
			} else if err != nil {
				srv.log.Debug("Read error", "err", err)
				slots <- struct{}{}
				return
			}
			break
		}

		remoteIP := netutil.AddrIP(fd.RemoteAddr())
		if err := srv.checkInboundConn(remoteIP); err != nil {
			srv.log.Debug("Rejected inbound connection", "addr", fd.RemoteAddr(), "err", err)
			fd.Close()
			slots <- struct{}{}
			continue
		}
		if remoteIP != nil {
			var addr *net.TCPAddr
			if tcp, ok := fd.RemoteAddr().(*net.TCPAddr); ok {
				addr = tcp
			}
			fd = newMeteredConn(fd, true, addr)
			srv.log.Trace("Accepted connection", "addr", fd.RemoteAddr())
		}
		go func() {
			///看来只要连接建立完成之后。 槽位就会归还。 SetupConn这个函数我们记得再dialTask.Do里面也有调用， 这个函数主要是执行连接的几次握手。
			srv.SetupConn(fd, inboundConn, nil)
			slots <- struct{}{}
		}()
	}
}

```
SetupConn,这个函数执行握手协议，并尝试把连接创建位一个peer对象。
```
func (srv *Server) setupConn(c *conn, flags connFlag, dialDest *enode.Node) error {
	// Prevent leftover pending conns from entering the handshake.
	srv.lock.Lock()
	running := srv.running
	srv.lock.Unlock()
	if !running {
		return errServerStopped
	}

	// If dialing, figure out the remote public key.
	if dialDest != nil {
		dialPubkey := new(ecdsa.PublicKey)
		if err := dialDest.Load((*enode.Secp256k1)(dialPubkey)); err != nil {
			err = errors.New("dial destination doesn't have a secp256k1 public key")
			srv.log.Trace("Setting up connection failed", "addr", c.fd.RemoteAddr(), "conn", c.flags, "err", err)
			return err
		}
	}

	// Run the RLPx handshake.
	////这里实际上执行的是rlpx.go里面的doEncHandshake.因为transport是conn的一个匿名字段。 匿名字段的方法会直接作为conn的一个方法
	remotePubkey, err := c.doEncHandshake(srv.PrivateKey)
	if err != nil {
		srv.log.Trace("Failed RLPx handshake", "addr", c.fd.RemoteAddr(), "conn", c.flags, "err", err)
		return err
	}
	// 如果连接握手的ID和对应的ID不匹配
	if dialDest != nil {
		c.node = dialDest
	} else {
		c.node = nodeFromConn(remotePubkey, c.fd)
	}
	clog := srv.log.New("id", c.node.ID(), "addr", c.fd.RemoteAddr(), "conn", c.flags)
	// 这个checkpoint其实就是把第一个参数发送给第二个参数指定的队列。然后从c.cout接收返回信息。 是一个同步的方法。
	//至于这里，后续的操作只是检查了一下连接是否合法就返回了。
	err = srv.checkpoint(c, srv.checkpointPostHandshake)
	if err != nil {
		clog.Trace("Rejected peer", "err", err)
		return err
	}

	// Run the capability negotiation handshake.
	phs, err := c.doProtoHandshake(srv.ourHandshake)
	if err != nil {
		clog.Trace("Failed p2p handshake", "err", err)
		return err
	}
	if id := c.node.ID(); !bytes.Equal(crypto.Keccak256(phs.ID), id[:]) {
		clog.Trace("Wrong devp2p handshake identity", "phsid", hex.EncodeToString(phs.ID))
		return DiscUnexpectedIdentity
	}
	c.caps, c.name = phs.Caps, phs.Name
	// 这里两次握手都已经完成了。 把c发送给checkpointAddPeer队列。 后台处理这个队列的时候，会处理这个连接
	err = srv.checkpoint(c, srv.checkpointAddPeer)
	if err != nil {
		clog.Trace("Rejected peer", "err", err)
		return err
	}

	return nil
}
```

上面说到的流程是listenLoop的流程，listenLoop主要是用来接收外部主动连接者的。 还有部分情况是节点需要主动发起连接来连接外部节点的流程。  以及处理刚才上面的checkpoint队列信息的流程。这部分代码都在server.run这个goroutine里面。



	func (srv *Server) run(dialstate dialer) {
		defer srv.loopWG.Done()
		var (
			peers        = make(map[discover.NodeID]*Peer)
			trusted      = make(map[discover.NodeID]bool, len(srv.TrustedNodes))
			taskdone     = make(chan task, maxActiveDialTasks)
			runningTasks []task
			queuedTasks  []task // tasks that can't run yet
		)
		// Put trusted nodes into a map to speed up checks.
		// Trusted peers are loaded on startup and cannot be
		// modified while the server is running.
		// 被信任的节点又这样一个特性， 如果连接太多，那么其他节点会被拒绝掉。但是被信任的节点会被接收。
		for _, n := range srv.TrustedNodes {
			trusted[n.ID] = true
		}
	
		// removes t from runningTasks
		// 定义了一个函数，用来从runningTasks队列删除某个Task
		delTask := func(t task) {
			for i := range runningTasks {
				if runningTasks[i] == t {
					runningTasks = append(runningTasks[:i], runningTasks[i+1:]...)
					break
				}
			}
		}
		// starts until max number of active tasks is satisfied
		// 同时开始连接的节点数量是16个。 遍历 runningTasks队列，并启动这些任务。
		startTasks := func(ts []task) (rest []task) {
			i := 0
			for ; len(runningTasks) < maxActiveDialTasks && i < len(ts); i++ {
				t := ts[i]
				log.Trace("New dial task", "task", t)
				go func() { t.Do(srv); taskdone <- t }()
				runningTasks = append(runningTasks, t)
			}
			return ts[i:]
		}
		scheduleTasks := func() {
			// Start from queue first.
			// 首先调用startTasks启动一部分，把剩下的返回给queuedTasks.
			queuedTasks = append(queuedTasks[:0], startTasks(queuedTasks)...)
			// Query dialer for new tasks and start as many as possible now.
			// 调用newTasks来生成任务，并尝试用startTasks启动。并把暂时无法启动的放入queuedTasks队列
			if len(runningTasks) < maxActiveDialTasks {
				nt := dialstate.newTasks(len(runningTasks)+len(queuedTasks), peers, time.Now())
				queuedTasks = append(queuedTasks, startTasks(nt)...)
			}
		}
	
	running:
		for {
			//调用 dialstate.newTasks来生成新任务。 并调用startTasks启动新任务。
			//如果 dialTask已经全部启动，那么会生成一个睡眠超时任务。
			scheduleTasks()
	
			select {
			case <-srv.quit:
				// The server was stopped. Run the cleanup logic.
				break running
			case n := <-srv.addstatic:
				// This channel is used by AddPeer to add to the
				// ephemeral static peer list. Add it to the dialer,
				// it will keep the node connected.
				log.Debug("Adding static node", "node", n)
				dialstate.addStatic(n)
			case n := <-srv.removestatic:
				// This channel is used by RemovePeer to send a
				// disconnect request to a peer and begin the
				// stop keeping the node connected
				log.Debug("Removing static node", "node", n)
				dialstate.removeStatic(n)
				if p, ok := peers[n.ID]; ok {
					p.Disconnect(DiscRequested)
				}
			case op := <-srv.peerOp:
				// This channel is used by Peers and PeerCount.
				op(peers)
				srv.peerOpDone <- struct{}{}
			case t := <-taskdone:
				// A task got done. Tell dialstate about it so it
				// can update its state and remove it from the active
				// tasks list.
				log.Trace("Dial task done", "task", t)
				dialstate.taskDone(t, time.Now())
				delTask(t)
			case c := <-srv.posthandshake:
				// A connection has passed the encryption handshake so
				// the remote identity is known (but hasn't been verified yet).
				// 记得之前调用checkpoint方法，会把连接发送给这个channel。
				if trusted[c.id] {
					// Ensure that the trusted flag is set before checking against MaxPeers.
					c.flags |= trustedConn
				}
				// TODO: track in-progress inbound node IDs (pre-Peer) to avoid dialing them.
				select {
				case c.cont <- srv.encHandshakeChecks(peers, c):
				case <-srv.quit:
					break running
				}
			case c := <-srv.addpeer:
				// At this point the connection is past the protocol handshake.
				// Its capabilities are known and the remote identity is verified.
				// 两次握手之后会调用checkpoint把连接发送到addpeer这个channel。
				// 然后通过newPeer创建了Peer对象。 
				// 启动一个goroutine 启动peer对象。 调用了peer.run方法。
				err := srv.protoHandshakeChecks(peers, c)
				if err == nil {
					// The handshakes are done and it passed all checks.
					p := newPeer(c, srv.Protocols)
					// If message events are enabled, pass the peerFeed
					// to the peer
					if srv.EnableMsgEvents {
						p.events = &srv.peerFeed
					}
					name := truncateName(c.name)
					log.Debug("Adding p2p peer", "id", c.id, "name", name, "addr", c.fd.RemoteAddr(), "peers", len(peers)+1)
					peers[c.id] = p
					go srv.runPeer(p)
				}
				// The dialer logic relies on the assumption that
				// dial tasks complete after the peer has been added or
				// discarded. Unblock the task last.
				select {
				case c.cont <- err:
				case <-srv.quit:
					break running
				}
			case pd := <-srv.delpeer:
				// A peer disconnected.
				d := common.PrettyDuration(mclock.Now() - pd.created)
				pd.log.Debug("Removing p2p peer", "duration", d, "peers", len(peers)-1, "req", pd.requested, "err", pd.err)
				delete(peers, pd.ID())
			}
		}
	
		log.Trace("P2P networking is spinning down")
	
		// Terminate discovery. If there is a running lookup it will terminate soon.
		if srv.ntab != nil {
			srv.ntab.Close()
		}
		if srv.DiscV5 != nil {
			srv.DiscV5.Close()
		}
		// Disconnect all peers.
		for _, p := range peers {
			p.Disconnect(DiscQuitting)
		}
		// Wait for peers to shut down. Pending connections and tasks are
		// not handled here and will terminate soon-ish because srv.quit
		// is closed.
		for len(peers) > 0 {
			p := <-srv.delpeer
			p.log.Trace("<-delpeer (spindown)", "remainingTasks", len(runningTasks))
			delete(peers, p.ID())
		}
	}


runPeer方法

	// runPeer runs in its own goroutine for each peer.
	// it waits until the Peer logic returns and removes
	// the peer.
	func (srv *Server) runPeer(p *Peer) {
		if srv.newPeerHook != nil {
			srv.newPeerHook(p)
		}
	
		// broadcast peer add
		srv.peerFeed.Send(&PeerEvent{
			Type: PeerEventTypeAdd,
			Peer: p.ID(),
		})
	
		// run the protocol
		remoteRequested, err := p.run()
	
		// broadcast peer drop
		srv.peerFeed.Send(&PeerEvent{
			Type:  PeerEventTypeDrop,
			Peer:  p.ID(),
			Error: err.Error(),
		})
	
		// Note: run waits for existing peers to be sent on srv.delpeer
		// before returning, so this send should not select on srv.quit.
		srv.delpeer <- peerDrop{p, err, remoteRequested}
	}


总结：

server对象主要完成的工作把之前介绍的所有组件组合在一起。 使用rlpx.go来处理加密链路。 使用discover来处理节点发现和查找。  使用dial来生成和连接需要连接的节点。 使用peer对象来处理每个连接。

server启动了一个listenLoop来监听和接收新的连接。 启动一个run的goroutine来调用dialstate生成新的dial任务并进行连接。 goroutine之间使用channel来进行通讯和配合。