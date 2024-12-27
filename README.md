app NextGenSocial {
    config {
        app_name = "NextGenSocial"
        database = "postgresql://username:password@localhost/socialdb"
        port = 8080
    }

    // ==== Database Schema ====
    model User {
        id: UUID @primary
        username: String @unique
        email: String @unique @encrypted
        password: String @hash
        profile_picture: String
        bio: String
        followers: List<UUID>
        created_at: Timestamp
    }

    model Post {
        id: UUID @primary
        user_id: UUID @foreign(User.id)
        content: String
        media_url: String @optional
        likes: Int @default(0)
        created_at: Timestamp
    }

    model Message {
        id: UUID @primary
        sender_id: UUID @foreign(User.id)
        receiver_id: UUID @foreign(User.id)
        content: String @encrypted
        sent_at: Timestamp
    }

    // ==== Backend Services ====
    service UserService {
        // User registration
        func register(username: String, email: String, password: String): User {
            return db.insert("users", {
                id: uuid(),
                username: username,
                email: email,
                password: hash(password),
                created_at: now()
            })
        }

        // Login authentication
        func login(email: String, password: String): Token {
            val user = db.queryOne("SELECT * FROM users WHERE email = ?", email)
            if (!checkHash(password, user.password)) {
                throw Error("Invalid credentials")
            }
            return auth.generateToken({ user_id: user.id })
        }
    }

    service PostService {
        // Create a new post
        func createPost(userId: UUID, content: String, mediaUrl: String? = null): Post {
            return db.insert("posts", {
                id: uuid(),
                user_id: userId,
                content: content,
                media_url: mediaUrl,
                created_at: now()
            })
        }

        // Fetch posts for a user feed
        func fetchFeed(userId: UUID): List<Post> {
            val followers = db.query("SELECT followers FROM users WHERE id = ?", userId)
            return db.query(
                "SELECT * FROM posts WHERE user_id IN ? ORDER BY created_at DESC LIMIT 50", followers
            )
        }
    }

    service ChatService {
        // Send a message
        func sendMessage(senderId: UUID, receiverId: UUID, content: String): Message {
            return db.insert("messages", {
                id: uuid(),
                sender_id: senderId,
                receiver_id: receiverId,
                content: encrypt(content),
                sent_at: now()
            })
        }

        // Fetch chat history
        func getChatHistory(userId: UUID, contactId: UUID): List<Message> {
            return db.query(
                "SELECT * FROM messages WHERE (sender_id = ? AND receiver_id = ?) OR (sender_id = ? AND receiver_id = ?)",
                userId, contactId, contactId, userId
            )
        }
    }

    // ==== Frontend Components ====
    component UserProfile {
        props userId: UUID
        state user: User

        async onLoad() {
            this.user = await api.fetch("/user/" + this.props.userId)
        }

        render() {
            return <div>
                <img src={this.user.profile_picture} class="profile-pic" />
                <h1>{this.user.username}</h1>
                <p>{this.user.bio}</p>
                <button onclick={() => followUser(this.user.id)}>Follow</button>
            </div>
        }
    }

    component Feed {
        state posts: List<Post>

        async onLoad() {
            this.posts = await api.fetch("/feed")
        }

        render() {
            return <div>
                {this.posts.map(post =>
                    <div class="post">
                        <h3>{post.user.username}</h3>
                        <p>{post.content}</p>
                        {post.media_url && <img src={post.media_url} />}
                        <button onclick={() => likePost(post.id)}>Like</button>
                    </div>
                )}
            </div>
        }
    }

    // ==== Routes ====
    route("/register", UserService.register)
    route("/login", UserService.login)
    route("/feed", PostService.fetchFeed)
    route("/post", PostService.createPost)
    route("/chat/send", ChatService.sendMessage)
    route("/chat/history", ChatService.getChatHistory)

    // ==== Real-Time Events ====
    event onPostLiked(postId: UUID) {
        val post = db.queryOne("SELECT * FROM posts WHERE id = ?", postId)
        post.likes += 1
        db.update("posts", post)
        broadcast("postLiked", post)
    }

    event onNewMessage(senderId: UUID, receiverId: UUID, content: String) {
        broadcastTo(receiverId, "newMessage", { senderId, content })
    }
}
