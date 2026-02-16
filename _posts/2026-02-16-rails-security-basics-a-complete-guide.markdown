---
layout: default
title: "Rails Security Basics: A Complete Guide to Built-in Security Features"
date: 2026-02-16 11:41:00 +0530
categories: engineering security
---

I read the Ruby on Rails book and figure out how Rails actually keeps things secure—what are the real security practices baked in?

At first, I thought, "Oh great, another boring security lecture." But as I dug into the official [Securing Rails Applications guide](https://guides.rubyonrails.org/security.html) and worked through *Agile Web Development with Rails 8* (especially the Depot app), I realized something amazing: Rails isn't just a framework—it's like having a security expert constantly watching your back.

The phrase "secure by default" gets thrown around a lot, but Rails genuinely lives up to it. Most of the scary OWASP Top 10 vulnerabilities? Rails handles them automatically if you don't fight the framework.

In this comprehensive guide, I'll walk you through everything I learned, with real examples you can use in your projects today. Whether you're building your first Rails app or you're a seasoned developer, understanding these security features will make you write better, safer code.

---

## Table of Contents

1. [Authentication Made Simple with Rails 8](#authentication)
2. [Session Management: Keeping Users Secure](#sessions)
3. [CSRF Protection: The Silent Guardian](#csrf)
4. [XSS Prevention: Trust No Input](#xss)
5. [SQL Injection: Rails Has Your Back](#sql-injection)
6. [Strong Parameters: No More Mass Assignment Nightmares](#strong-parameters)
7. [HTTP Headers and Security Configuration](#http-headers)
8. [Advanced Security Topics](#advanced-topics)
9. [Security Checklist and Best Practices](#checklist)

---

## 1. Authentication Made Simple with Rails 8 {#authentication}

### The Old Way vs. The Rails 8 Way

Remember the days of adding Devise, configuring it for hours, and still not being 100% sure it was set up correctly? Or rolling your own authentication and worrying about all the edge cases?

Rails 8 changes everything with its built-in authentication generator.

### Getting Started: One Command to Rule Them All

```bash
# That's it. Seriously.
bin/rails generate authentication
bin/rails db:migrate
```

This single command gives you:

- A complete `User` model with secure password handling
- A `Session` model for tracking logins
- Controller logic for registration, login, and logout
- Password reset functionality with secure tokens
- Email verification (optional but recommended)

### Understanding `has_secure_password`

Let's look at what the generator creates for your User model:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  
  # This one line gives you:
  # - password and password_confirmation attributes (virtual, not stored)
  # - Automatic bcrypt hashing (never stores plain text passwords)
  # - Validations: password presence, confirmation matching, max 72 bytes
  # - authenticate(password) method for login checks
  # - authenticate_by class method for safe lookups
end
```

### Why Bcrypt Matters

Bcrypt isn't just a hashing algorithm—it's specifically designed to be slow. That sounds bad, right? Wrong! Here's why:

```ruby
# Bad approach (NEVER DO THIS):
user.password_digest = Digest::SHA256.hexdigest(password)
# Problem: SHA256 is FAST. Attackers can try billions of passwords per second.

# Good approach (Rails does this automatically):
user.password = "secure_password"
# Uses bcrypt which is intentionally slow (configurable "cost")
# An attacker might only try thousands per second instead of billions
# Your legitimate user notices no difference (one hash takes ~100ms)
```

### Real-World Example: Building a Login System

Here's how the authentication is typically implemented:

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  def create
    # authenticate_by is Rails 8's secure way to check credentials
    # It's timing-attack resistant (always takes same time whether user exists or not)
    if user = User.authenticate_by(
      email_address: params[:email_address],
      password: params[:password]
    )
      # CRITICAL: reset_session prevents session fixation attacks
      # This creates a new session ID, invalidating any old ones
      reset_session
      
      # Now store the user_id in the new, clean session
      session[:user_id] = user.id
      
      redirect_to root_path, notice: "Welcome back!"
    else
      # Don't tell attackers whether the email exists or password was wrong
      flash.now[:alert] = "Invalid email or password"
      render :new, status: :unprocessable_entity
    end
  end
  
  def destroy
    # Always reset session on logout too
    reset_session
    redirect_to root_path, notice: "Logged out successfully"
  end
end
```

### Password Reset Flow (The Secure Way)

Password resets are tricky. You need to verify someone's identity without their password. Here's how Rails 8's generator handles it:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  
  # Generate a secure random token for password reset
  def generate_password_reset_token
    # signed_id creates a cryptographically signed token
    # It expires after 15 minutes
    # It can't be forged or tampered with
    signed_id(purpose: :password_reset, expires_in: 15.minutes)
  end
  
  # Verify and find user by token
  def self.find_by_password_reset_token(token)
    # This automatically validates:
    # - Token hasn't been tampered with
    # - Token hasn't expired
    # - Purpose matches
    find_signed(token, purpose: :password_reset)
  end
end

# app/controllers/password_resets_controller.rb
class PasswordResetsController < ApplicationController
  def create
    user = User.find_by(email_address: params[:email_address])
    
    if user
      token = user.generate_password_reset_token
      
      # Send email with reset link
      PasswordResetMailer.reset(user, token).deliver_later
    end
    
    # Always show same message (don't reveal if email exists)
    redirect_to root_path, 
      notice: "If that email exists, you'll receive reset instructions"
  end
  
  def update
    # Find user by token (automatically validates expiry and signature)
    user = User.find_by_password_reset_token(params[:token])
    
    if user && user.update(password_params)
      # Password changed successfully
      reset_session
      session[:user_id] = user.id
      redirect_to root_path, notice: "Password changed successfully"
    else
      # Token invalid, expired, or password validation failed
      render :edit, status: :unprocessable_entity
    end
  end
end
```

### Building an Authorization System

Authentication tells you WHO someone is. Authorization tells you what they can DO. Here's a basic example:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  private
  
  def current_user
    # Memoization pattern: only query database once per request
    @current_user ||= User.find_by(id: session[:user_id]) if session[:user_id]
  end
  helper_method :current_user  # Make available in views
  
  def require_authentication
    unless current_user
      redirect_to login_path, alert: "Please log in"
    end
  end
  
  def require_admin
    unless current_user&.admin?
      redirect_to root_path, alert: "Access denied"
    end
  end
end

# app/controllers/admin/products_controller.rb
class Admin::ProductsController < ApplicationController
  before_action :require_admin  # Protect all admin actions
  
  def index
    @products = Product.all
  end
  
  # ... other admin actions
end
```

### View Helpers for Conditional UI

```erb
<!-- app/views/layouts/application.html.erb -->
<nav>
  <%= link_to "Home", root_path %>
  
  <% if current_user %>
    <!-- Only show these links to logged-in users -->
    <%= link_to "My Account", account_path %>
    
    <% if current_user.admin? %>
      <!-- Only show admin links to admins -->
      <%= link_to "Admin Panel", admin_root_path %>
    <% end %>
    
    <%= button_to "Log Out", logout_path, method: :delete %>
  <% else %>
    <%= link_to "Log In", login_path %>
    <%= link_to "Sign Up", signup_path %>
  <% end %>
</nav>
```

### Key Takeaways: Authentication

1. **Use the generator**: Don't reinvent the wheel.
2. **Always call `reset_session`**: On login AND logout.
3. **Use `authenticate_by`**: It's timing-attack resistant.
4. **Implement password resets securely**: Use signed tokens with expiry.
5. **Separate authentication from authorization**: Know the difference.
6. **Never reveal if emails exist**: Use the same message for all reset requests.

---

## 2. Session Management: Keeping Users Secure {#sessions}

### Understanding Rails Sessions

Sessions are how Rails remembers who you are between requests. HTTP is stateless, so without sessions, every request would be anonymous.

### The Cookie Store (Rails Default)

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store,
  key: '_myapp_session',
  expire_after: 2.weeks,
  secure: Rails.env.production?,    # Only send over HTTPS in production
  httponly: true,                   # JavaScript can't access this cookie
  same_site: :lax                   # CSRF protection at cookie level
```

**What's happening here?**

- **Encrypted Cookies**: Session data is encrypted using your `secret_key_base`.
- **Signed Cookies**: Even if someone decrypts it, they can't modify it without detection.
- **HttpOnly**: JavaScript can't steal your session via XSS attacks.
- **Secure**: In production, cookies only sent over HTTPS.
- **SameSite**: Browser won't send cookie in cross-site requests (CSRF protection).

### Session Size Limits

Cookies have a 4KB limit. Don't store large objects:

```ruby
# BAD: Storing too much in session
session[:shopping_cart] = {
  items: 50.times.map { |i| { id: i, details: "..." } }
}

# GOOD: Store just the IDs
session[:cart_item_ids] = [1, 2, 3, 4, 5]

# Then in your controller:
@cart_items = Product.find(session[:cart_item_ids])
```

### Session Fixation: The Attack and Defense

**The Attack:**

1. Attacker creates a session and gets session ID `ABC123`.
2. Attacker tricks victim into using that session (via link or XSS).
3. Victim logs in using the attacker's session ID.
4. Attacker now has access to victim's authenticated session.

**The Defense:**

```ruby
# app/controllers/sessions_controller.rb
def create
  user = User.authenticate_by(email_address: params[:email], password: params[:password])
  
  if user
    # THIS LINE IS CRITICAL
    # It generates a new session ID, making the old one worthless
    reset_session
    
    session[:user_id] = user.id  # Now store in the NEW session
    redirect_to dashboard_path
  end
end
```

### Database-Backed Sessions (For Advanced Cases)

Sometimes you need more than cookies can offer:

```ruby
# Gemfile
gem 'activerecord-session_store'

# Generate session table
rails generate active_record:session_migration
rails db:migrate

# config/initializers/session_store.rb
Rails.application.config.session_store :active_record_store,
  key: '_myapp_session',
  expire_after: 2.weeks
```

**When to use database sessions:**

- Need to store more than 4KB of data.
- Want to forcefully invalidate sessions (logout from all devices).
- Need audit trails of session activity.
- Want to limit concurrent sessions per user.

### Session Cleanup Task

If using database sessions, clean up old ones:

```ruby
# lib/tasks/sessions.rake
namespace :sessions do
  desc "Clean up expired sessions"
  task cleanup: :environment do
    # Delete sessions not updated in last hour
    ActiveRecord::SessionStore::Session
      .where("updated_at < ?", 1.hour.ago)
      .delete_all
    
    puts "Expired sessions cleaned up"
  end
end
```

### Real-World Session Example: Shopping Cart

```ruby
# app/controllers/cart_controller.rb
class CartController < ApplicationController
  before_action :initialize_cart
  
  def add
    product = Product.find(params[:product_id])
    @cart[:items] << product.id
    
    # Be careful with session size!
    session[:cart] = @cart
    
    redirect_to cart_path, notice: "Added #{product.name} to cart"
  end
  
  def clear
    # Clear cart from session
    session.delete(:cart)
    redirect_to root_path, notice: "Cart cleared"
  end
  
  private
  
  def initialize_cart
    # Initialize empty cart if doesn't exist
    session[:cart] ||= { items: [] }
    @cart = session[:cart]
  end
end
```

### Key Takeaways: Sessions

1. **Rails encrypts and signs session cookies**: Can't be read or tampered with.
2. **Always `reset_session` on login/logout**: Prevents fixation attacks.
3. **Use HttpOnly and Secure flags**: Automatic in production.
4. **Keep sessions small**: Under 4KB for cookie store.
5. **Consider database sessions**: For advanced needs.
6. **Clean up old sessions**: If using database store.

---

## 3. CSRF Protection: The Silent Guardian {#csrf}

### What is CSRF (Cross-Site Request Forgery)?

Imagine this scenario:

1. You log into your bank.
2. You visit a malicious site (in another tab).
3. That site has a hidden form that submits to your bank: "Transfer $1000 to attacker".
4. Because you're logged in, the browser sends your session cookie.
5. Bank sees authenticated request and processes it.

**This is CSRF.** And Rails protects you automatically.

### How Rails Prevents CSRF

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # This is enabled by DEFAULT in Rails
  protect_from_forgery with: :exception
  
  # Other options:
  # protect_from_forgery with: :null_session  # Clear session if invalid
  # protect_from_forgery with: :reset_session # Create new session if invalid
end
```

### The Magic Token

Every form Rails generates includes a hidden token:

```erb
<%= form_with model: @post do |f| %>
  <%= f.text_field :title %>
  <%= f.text_area :body %>
  <%= f.submit %>
<% end %>
```

**Generated HTML:**

```html
<form action="/posts" method="post">
  <!-- THIS is the CSRF token -->
  <input type="hidden" 
         name="authenticity_token" 
         value="long_random_string_that_changes_per_session">
  
  <input type="text" name="post[title]">
  <textarea name="post[body]"></textarea>
  <input type="submit" value="Create Post">
</form>
```

Rails verifies this token on EVERY non-GET request. If it's missing or invalid, the request is rejected.

### CSRF Protection for JavaScript/AJAX

Modern apps use fetch() or Axios for AJAX requests. Rails handles this too:

```erb
<!-- app/views/layouts/application.html.erb -->
<head>
  <title>My App</title>
  <%= csrf_meta_tags %>
  <!-- This generates: -->
  <!-- <meta name="csrf-param" content="authenticity_token"> -->
  <!-- <meta name="csrf-token" content="long_random_token"> -->
</head>
```

**In your JavaScript:**

```javascript
// Vanilla JavaScript with fetch
const token = document.querySelector('meta[name="csrf-token"]').content;

fetch('/posts', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': token  // Include the token in headers
  },
  body: JSON.stringify({ post: { title: 'New Post', body: 'Content' } })
});

// Or with Axios (set it globally)
axios.defaults.headers.common['X-CSRF-Token'] = token;

// Then just make requests normally
axios.post('/posts', { post: { title: 'New Post' } });
```

### Turbo and CSRF (Rails 7+)

If you're using Hotwire/Turbo (default in Rails 7+), it handles CSRF tokens automatically:

```erb
<!-- Just include csrf_meta_tags, Turbo does the rest -->
<%= csrf_meta_tags %>

<!-- Your Turbo Frames and Streams work securely out of the box -->
<%= turbo_frame_tag "post_#{@post.id}" do %>
  <%= form_with model: @post do |f| %>
    <!-- Token included automatically by Turbo -->
  <% end %>
<% end %>
```

### API Mode: When to Skip CSRF

Building a pure API (no browser sessions)? You can skip CSRF:

```ruby
# app/controllers/api/base_controller.rb
class Api::BaseController < ActionController::API
  # ActionController::API doesn't include session/cookie middleware
  # No sessions = no CSRF concerns
  
  # Use token-based auth instead (JWT, OAuth, API keys)
end
```

**But be careful:**

```ruby
# If your API uses cookies/sessions for auth, KEEP CSRF protection
class Api::BaseController < ActionController::Base
  protect_from_forgery with: :null_session
  # This allows both token and session auth
end
```

### Testing CSRF Protection

```ruby
# test/controllers/posts_controller_test.rb
class PostsControllerTest < ActionDispatch::IntegrationTest
  test "rejects POST without CSRF token" do
    # By default, tests include token
    # Use `post` directly without Rails helpers to skip it
    post posts_url, 
         params: { post: { title: "Test" } },
         headers: { 'Content-Type': 'application/json' }
    
    # Should be rejected
    assert_response :unprocessable_entity
  end
  
  test "accepts POST with valid CSRF token" do
    # Using form helpers automatically includes token
    post posts_url, params: { post: { title: "Test" } }
    assert_response :redirect
  end
end
```

### Real-World Example: AJAX Form Submission

```erb
<!-- app/views/comments/_form.html.erb -->
<div data-controller="comment-form">
  <%= form_with model: [@post, @comment], 
                data: { action: "submit->comment-form#submit" } do |f| %>
    <%= f.text_area :body, data: { comment_form_target: "body" } %>
    <%= f.submit "Post Comment" %>
  <% end %>
  
  <div data-comment-form-target="errors"></div>
</div>
```

```javascript
// app/javascript/controllers/comment_form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["body", "errors"]
  
  async submit(event) {
    event.preventDefault()
    
    const form = event.target
    const formData = new FormData(form)
    
    // Get CSRF token from meta tag
    const token = document.querySelector('meta[name="csrf-token"]').content
    
    try {
      const response = await fetch(form.action, {
        method: 'POST',
        headers: {
          'X-CSRF-Token': token,
          'Accept': 'application/json'
        },
        body: formData
      })
      
      if (response.ok) {
        const data = await response.json()
        this.bodyTarget.value = ''  // Clear form
        // Maybe add comment to page dynamically
      } else {
        const errors = await response.json()
        this.errorsTarget.textContent = errors.join(', ')
      }
    } catch (error) {
      console.error('Error:', error)
    }
  }
}
```

### Key Takeaways: CSRF

1. **It's automatic**: `protect_from_forgery` is on by default.
2. **Always include csrf_meta_tags**: In your layout's `<head>`.
3. **Forms include tokens automatically**: Using `form_with` or `form_for`.
4. **AJAX needs manual token**: Include `X-CSRF-Token` header.
5. **Turbo handles it for you**: If using Hotwire.
6. **APIs might not need it**: If using token auth instead of sessions.

---

## 4. XSS Prevention: Trust No Input {#xss}

### What is XSS (Cross-Site Scripting)?

XSS is when an attacker injects malicious JavaScript into your pages that runs in other users' browsers.

**Example attack:**

```ruby
# User submits a comment
comment.body = "<script>steal_cookies()</script>"

# Your view renders it
<div class="comment">
  <%= raw comment.body %>  # DANGEROUS!
</div>

# Now every visitor runs that malicious script
```

### Rails' Automatic Escaping: Your First Line of Defense

Good news: Rails escapes HTML by default!

```erb
<!-- Input: comment.body = "<script>alert('XSS')</script>" -->

<!-- Safe (default behavior): -->
<div><%= comment.body %></div>
<!-- Output: &lt;script&gt;alert('XSS')&lt;/script&gt; -->
<!-- Browser shows text, doesn't execute script -->

<!-- Dangerous (explicitly bypassing safety): -->
<div><%= raw comment.body %></div>
<div><%= comment.body.html_safe %></div>
<!-- Output: <script>alert('XSS')</script> -->
<!-- Browser EXECUTES the script! -->
```

### The Golden Rule: Never Trust User Input

```ruby
# DANGEROUS: Trusting user input
def show
  @message = params[:message].html_safe  # NEVER DO THIS
end

# View
<div><%= @message %></div>  # Will execute any scripts


# SAFE: Let Rails escape it
def show
  @message = params[:message]  # No .html_safe
end

# View
<div><%= @message %></div>  # Scripts will be escaped
```

### When You Need HTML: Use the Sanitize Helper

Sometimes users SHOULD be able to format text (bold, italics, links). Use `sanitize`:

```erb
<!-- Allow only safe HTML tags -->
<div class="comment-body">
  <%= sanitize comment.body, 
      tags: %w(p br strong em a ul ol li),
      attributes: %w(href title) %>
</div>
```

**Example:**

```ruby
comment.body = "
  <p>This is <strong>bold</strong> text.</p>
  <script>alert('XSS')</script>
  <a href='javascript:evil()'>Click me</a>
  <a href='https://example.com'>Safe link</a>
"

sanitize(comment.body, tags: %w(p strong a), attributes: %w(href))
# Result:
# <p>This is <strong>bold</strong> text.</p>
# alert('XSS')
# Click me
# <a href="https://example.com">Safe link</a>
```

Notice:
- `<script>` tag removed completely
- `javascript:` URL removed (dangerous)
- Safe HTML kept intact

### Building a Rich Text Editor (Action Text)

Rails 7+ includes Action Text for rich content:

```ruby
# Add Action Text to your model
class Post < ApplicationRecord
  has_rich_text :body
end

# Migration (generated automatically)
rails action_text:install
```

```erb
<!-- app/views/posts/_form.html.erb -->
<%= form_with model: @post do |f| %>
  <%= f.label :title %>
  <%= f.text_field :title %>
  
  <%= f.label :body %>
  <%= f.rich_text_area :body %>
  <!-- Includes Trix editor with safe HTML sanitization built-in -->
  
  <%= f.submit %>
<% end %>

<!-- app/views/posts/show.html.erb -->
<h1><%= @post.title %></h1>
<div class="post-body">
  <%= @post.body %>
  <!-- Automatically sanitized, safe to display -->
</div>
```

Action Text automatically:
- Sanitizes HTML
- Allows only safe tags
- Stores attachments securely
- Handles embedded media

### Custom Sanitization Rules

```ruby
# config/initializers/sanitize.rb
class CustomSanitizer < Rails::Html::SafeListSanitizer
  def allowed_tags
    super + %w(table thead tbody tr th td)
  end
  
  def allowed_attributes
    super + %w(colspan rowspan class)
  end
end

# Use in your views
<%= sanitize(user_content, scrubber: CustomSanitizer.new) %>
```

### Content Security Policy (CSP): The Nuclear Option

CSP is a browser security feature that blocks unauthorized scripts:

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  # Only allow scripts from your domain
  policy.script_src :self
  
  # Allow inline styles (needed for many libraries)
  policy.style_src :self, :unsafe_inline
  
  # Allow images from anywhere
  policy.img_src :self, :data, "https:"
  
  # Only allow AJAX to your domain
  policy.connect_src :self
  
  # Block object/embed tags
  policy.object_src :none
  
  # Specify where forms can submit
  policy.form_action :self
  
  # Enable report-only mode first to test
  # policy.report_uri "/csp-violation-report-endpoint"
end

# Enforce in production only
Rails.application.config.content_security_policy_nonce_generator = 
  ->(request) { SecureRandom.base64(16) }

Rails.application.config.content_security_policy_nonce_directives = 
  %w(script-src style-src)
```

**What CSP does:**

Even if an attacker injects a `<script>` tag, the browser refuses to run it because it's not from an allowed source.

### XSS in JavaScript Context

Be especially careful when outputting user data in JavaScript:

```erb
<!-- DANGEROUS -->
<script>
  var username = "<%= @user.name %>";
  // If name is: "; alert('XSS'); //"
  // Becomes: var username = ""; alert('XSS'); //"";
</script>

<!-- SAFE: Use JSON escaping -->
<script>
  var username = <%= @user.name.to_json %>;
  // Properly escaped as JSON string
</script>

<!-- EVEN BETTER: Use data attributes -->
<div id="user-info" data-username="<%= @user.name %>"></div>
<script>
  var username = document.getElementById('user-info').dataset.username;
  // Browser handles escaping automatically
</script>
```

### Real-World Example: Comment System

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
  belongs_to :user
  
  validates :body, presence: true, length: { maximum: 5000 }
  
  # Method to get safely formatted HTML
  def formatted_body
    # Convert markdown to HTML, then sanitize
    markdown = Redcarpet::Markdown.new(
      Redcarpet::Render::HTML,
      autolink: true,
      no_intra_emphasis: true
    )
    
    html = markdown.render(body)
    
    # Sanitize to allow only safe tags
    ActionController::Base.helpers.sanitize(
      html,
      tags: %w(p br strong em a ul ol li blockquote code pre),
      attributes: %w(href title)
    )
  end
end

# app/views/comments/_comment.html.erb
<div class="comment">
  <div class="comment-author">
    <%= comment.user.name %>  <!-- Automatically escaped -->
  </div>
  <div class="comment-body">
    <%= comment.formatted_body.html_safe %>
    <!-- .html_safe is OK here because formatted_body sanitizes -->
  </div>
</div>
```

### Testing for XSS

```ruby
# test/models/comment_test.rb
class CommentTest < ActiveSupport::TestCase
  test "sanitizes malicious scripts" do
    comment = Comment.create(
      body: "Hello <script>alert('XSS')</script>world",
      user: users(:one),
      post: posts(:one)
    )
    
    assert_not_includes comment.formatted_body, "<script>"
    assert_includes comment.formatted_body, "Hello"
    assert_includes comment.formatted_body, "world"
  end
  
  test "allows safe HTML" do
    comment = Comment.create(
      body: "This is **bold** and this is [a link](https://example.com)",
      user: users(:one),
      post: posts(:one)
    )
    
    assert_includes comment.formatted_body, "<strong>bold</strong>"
    assert_includes comment.formatted_body, '<a href="https://example.com">'
  end
end
```

### Key Takeaways: XSS

1. **Never use `raw` or `html_safe` on user input**: Rails escapes by default for a reason.
2. **Use `sanitize` for formatted text**: Whitelist allowed tags.
3. **Consider Action Text**: For rich content with built-in security.
4. **Implement CSP**: Extra layer of protection.
5. **Be careful in JavaScript**: Use `to_json` for data in scripts.
6. **Test sanitization**: Make sure malicious input is neutralized.

---

## 5. SQL Injection: Rails Has Your Back {#sql-injection}

### What is SQL Injection?

SQL injection is when an attacker manipulates your database queries by injecting malicious SQL code.

**Classic example:**

```ruby
# DANGEROUS: String interpolation
User.where("email = '#{params[:email]}'")

# If attacker sends: params[:email] = "' OR '1'='1"
# Query becomes: SELECT * FROM users WHERE email = '' OR '1'='1'
# Returns ALL users! (because '1'='1' is always true)

# Even worse:
params[:email] = "'; DROP TABLE users; --"
# Goodbye, users table!
```

### Rails' Safe Query Methods

Active Record provides several safe ways to query:

#### 1. Hash Conditions (Safest and Simplest)

```ruby
# SAFE: Rails parameterizes automatically
User.where(email: params[:email])
User.where(email: params[:email], active: true)
User.where(status: ['active', 'pending'])

# Generated SQL (PostgreSQL example):
# SELECT * FROM users WHERE email = $1
# Rails sends params[:email] as a separate parameter
# Database treats it as DATA, never as CODE
```

#### 2. Array Conditions (For Complex Queries)

```ruby
# SAFE: Use ? placeholders
User.where("email = ? AND created_at > ?", params[:email], 1.week.ago)

# SAFE: Named placeholders (more readable)
User.where(
  "email = :email AND status = :status",
  email: params[:email],
  status: params[:status]
)

# Still SAFE: Works with arrays
User.where("status IN (?)", ['active', 'pending'])
# Handles the SQL syntax for IN clauses correctly
```

### Common SQL Injection Mistakes

```ruby
# MISTAKE 1: String interpolation
# NEVER DO THIS
User.where("name = '#{params[:name]}'")  # VULNERABLE!

# FIX: Use placeholders
User.where("name = ?", params[:name])  # SAFE


# MISTAKE 2: Using ORDER BY with user input
# DANGEROUS
Post.order("#{params[:sort_by]} #{params[:direction]}")
# Attacker: ?sort_by=id;DROP TABLE posts;--

# FIX: Whitelist allowed columns
allowed_sort = ['created_at', 'title', 'author']
sort_by = allowed_sort.include?(params[:sort_by]) ? params[:sort_by] : 'created_at'
direction = %w[asc desc].include?(params[:direction]) ? params[:direction] : 'desc'
Post.order("#{sort_by} #{direction}")  # SAFE (whitelisted)
```

### Using Scopes Safely

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  # Safe scopes that use parameterized queries
  scope :by_category, ->(category) { where(category: category) }
  scope :in_price_range, ->(min, max) { 
    where("price BETWEEN ? AND ?", min, max) 
  }
  scope :search, ->(term) {
    sanitized = term.gsub(/[%_]/, '\\\\\0')
    where("name ILIKE ? OR description ILIKE ?", "%#{sanitized}%", "%#{sanitized}%")
  }
end

# Controller usage
@products = Product.all
@products = @products.by_category(params[:category]) if params[:category].present?
@products = @products.search(params[:q]) if params[:q].present?
```

### Key Takeaways: SQL Injection

1. **Never use string interpolation**: Always use `?` or `:named` placeholders.
2. **Hash conditions are safest**: For simple queries, use `where(column: value)`.
3. **Whitelist for ORDER BY**: User shouldn't control column names directly.
4. **Sanitize LIKE wildcards**: Escape `%` and `_` characters.
5. **Use scopes for reusable queries**: Encapsulates safe query logic.
6. **Test with malicious input**: Make sure injection attempts fail safely.

---

## 6. Strong Parameters: No More Mass Assignment Nightmares {#strong-parameters}

### The Mass Assignment Problem

Before Strong Parameters (Rails 4+), this was a major security hole:

```ruby
# OLD VULNERABLE CODE (Rails 3 and earlier)
class UsersController < ApplicationController
  def create
    @user = User.new(params[:user])  # DANGEROUS!
    @user.save
  end
end

# Attacker sends:
# POST /users
# {
#   user: {
#     name: "Evil User",
#     email: "evil@example.com",
#     admin: true  # ← Attacker makes themselves admin!
#   }
# }
```

### How Strong Parameters Work

Strong Parameters force you to explicitly whitelist allowed attributes:

```ruby
# SAFE: Modern Rails approach
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)  # Only allowed params
    
    if @user.save
      redirect_to @user, notice: 'User created'
    else
      render :new
    end
  end
  
  def update
    @user = User.find(params[:id])
    
    if @user.update(user_params)
      redirect_to @user, notice: 'User updated'
    else
      render :edit
    end
  end
  
  private
  
  # Whitelist pattern: require + permit
  def user_params
    params.require(:user).permit(:name, :email, :password, :password_confirmation)
    # admin is NOT in the list, so it can't be set via mass assignment
  end
end
```

### Nested Attributes

Handling forms with associations:

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments
  accepts_nested_attributes_for :comments
end

# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)
    @post.save
  end
  
  private
  
  def post_params
    params.require(:post).permit(
      :title,
      :body,
      :published,
      # Allow nested comments_attributes
      comments_attributes: [:id, :body, :author_name, :_destroy]
      # :id - needed for updating existing comments
      # :_destroy - needed for deleting comments
    )
  end
end
```

### Dynamic Permissions Based on User Role

```ruby
class PostsController < ApplicationController
  def update
    @post = Post.find(params[:id])
    
    if @post.update(post_params)
      redirect_to @post
    else
      render :edit
    end
  end
  
  private
  
  def post_params
    # Base attributes everyone can set
    allowed = [:title, :body, :tag_list]
    
    # Admins can set additional attributes
    if current_user.admin?
      allowed += [:featured, :published_at, :author_id]
    end
    
    # Authors can publish their own posts
    if current_user.id == @post.author_id
      allowed += [:published]
    end
    
    params.require(:post).permit(*allowed)
  end
end
```

### Key Takeaways: Strong Parameters

1. **Always use strong parameters**: Never pass `params` directly to models.
2. **Whitelist, don't blacklist**: Explicitly permit allowed attributes.
3. **Include :id and :_destroy**: For nested attributes that update or delete.
4. **Dynamic permissions**: Adjust allowed params based on user role.
5. **Never use .permit!**: It defeats the entire security mechanism.
6. **Test thoroughly**: Verify unpermitted params are filtered.

---

## 7. HTTP Headers and Security Configuration {#http-headers}

### Understanding Security Headers

HTTP headers are invisible to users but critical for security. They tell browsers how to handle your content.

### Rails Default Security Headers

Rails sets these automatically in production:

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    # These are the defaults in Rails 7+
    
    # Prevents clickjacking attacks
    config.action_dispatch.default_headers = {
      'X-Frame-Options' => 'SAMEORIGIN',  # Can't embed in external iframes
      'X-Content-Type-Options' => 'nosniff',  # Don't guess content types
      'X-XSS-Protection' => '0',  # Disabled (CSP is better)
      'Referrer-Policy' => 'strict-origin-when-cross-origin'  # Privacy
    }
  end
end
```

### HTTPS and HSTS (HTTP Strict Transport Security)

Always use HTTPS in production:

```ruby
# config/environments/production.rb
Rails.application.configure do
  # Force all connections over HTTPS
  config.force_ssl = true
  
  # This automatically:
  # - Redirects HTTP to HTTPS
  # - Sets Secure flag on cookies
  # - Enables HSTS
end
```

### Content Security Policy (CSP) In-Depth

CSP is your strongest defense against XSS:

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  # Where scripts can load from
  policy.script_src :self, :https  # Only your domain and HTTPS sources
  
  # Where stylesheets can load from
  policy.style_src :self, :https, :unsafe_inline  # Unsafe inline needed for many gems
  
  # Where images can load from
  policy.img_src :self, :https, :data  # :data for inline base64 images
  
  # Where fonts can load from
  policy.font_src :self, :https
  
  # Where AJAX/WebSocket can connect to
  policy.connect_src :self, 'wss:'  # WebSocket secure connections
  
  # Where media (audio/video) can load from
  policy.media_src :self
  
  # Where <object>, <embed>, <applet> can load from
  policy.object_src :none
  
  # Where forms can submit to
  policy.form_action :self
  
  # Where frames can load from
  policy.frame_src :self
end
```

### Rate Limiting (New in Rails 7.1)

Built-in rate limiting to prevent abuse:

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  # Limit login attempts
  rate_limit to: 5, within: 1.minute, only: :create, with: -> {
    redirect_to login_path, alert: "Too many login attempts. Try again later."
  }
end
```

### Key Takeaways: HTTP Headers

1. **Always use HTTPS in production**: `config.force_ssl = true`.
2. **Enable HSTS**: Prevents downgrade attacks.
3. **Implement CSP**: Strongest XSS protection.
4. **Use Permissions Policy**: Restrict browser features.
5. **Configure CORS carefully**: Only allow trusted origins.
6. **Add rate limiting**: Prevent abuse and DOS attacks.
7. **Test your headers**: Verify they're set correctly.

---

## 8. Advanced Security Topics {#advanced-topics}

### Secrets Management

Never commit secrets to version control:

```ruby
# config/credentials.yml.enc (encrypted)
# Edit with: rails credentials:edit

# Production credentials
secret_key_base: [long random string]

aws:
  access_key_id: AKIAI...
  secret_access_key: wJalr...
  bucket: my-app-production

stripe:
  publishable_key: pk_live_...
  secret_key: sk_live_...
```

### Timing Attacks and Safe Comparisons

Timing attacks exploit the fact that string comparison stops at first difference:

```ruby
# VULNERABLE: Timing attack possible
def check_api_key(provided_key)
  if provided_key == VALID_API_KEY  # Comparison time varies!
    true
  else
    false
  end
end

# An attacker can measure response time:
# "a" != "secret" - fast (fails at first char)
# "s" != "secret" - slightly slower (fails at second char)
# "se" != "secret" - even slower...
# Eventually they reconstruct the key!


# SAFE: Constant-time comparison
def check_api_key(provided_key)
  ActiveSupport::SecurityUtils.secure_compare(provided_key, VALID_API_KEY)
end

# This always takes the same time regardless of where strings differ
```

### Dependency Security Scanning

Automatically check for vulnerable gems:

```bash
# bundler-audit: Check Gemfile.lock against CVE database
gem install bundler-audit

# Check for vulnerabilities
bundle audit check

# Brakeman: Static analysis security scanner
gem install brakeman

# Scan your Rails app
brakeman
```

### Key Takeaways: Advanced Topics

1. **Never commit secrets**: Use `credentials.yml.enc` or env vars.
2. **Use constant-time comparisons**: For tokens and keys.
3. **Validate uploads by content**: Not just extension.
4. **Scan for malware**: If accepting uploads.
5. **Audit sensitive actions**: Track who did what.
6. **Scan dependencies regularly**: Use bundler-audit and Brakeman.

---

## 9. Security Checklist and Best Practices {#checklist}

### Pre-Deployment Checklist

Before pushing to production, verify:

**Authentication & Authorization**
- [ ] Using Rails 8 authentication generator or equivalent
- [ ] Passwords hashed with bcrypt (via `has_secure_password`)
- [ ] `reset_session` called on login and logout
- [ ] Password reset tokens expire (< 1 hour)
- [ ] Authorization checks on all admin/sensitive actions
- [ ] No sensitive data in logs (passwords, tokens, etc.)

**Session Security**
- [ ] Sessions encrypted and signed (default in Rails)
- [ ] HttpOnly cookies enabled (default)
- [ ] Secure flag on cookies in production (`force_ssl = true`)
- [ ] SameSite cookie attribute set
- [ ] Session timeout configured appropriately

**CSRF Protection**
- [ ] `protect_from_forgery` enabled (default)
- [ ] `csrf_meta_tags` in layout head
- [ ] AJAX requests include CSRF token
- [ ] Non-GET API endpoints verify token or use alternative auth

**XSS Prevention**
- [ ] Never use `raw` or `.html_safe` on user input
- [ ] Using `sanitize` helper for formatted text
- [ ] Content Security Policy configured

**SQL Injection Protection**
- [ ] No string interpolation in queries
- [ ] Using hash conditions or `?` placeholders
- [ ] ORDER BY columns whitelisted
- [ ] LIKE wildcards escaped

**Strong Parameters**
- [ ] All controller actions use strong parameters
- [ ] No use of `.permit!` (permits everything)

**HTTP Security**
- [ ] HTTPS enforced (`force_ssl = true`)
- [ ] HSTS enabled with long max-age
- [ ] Security headers configured (X-Frame-Options, etc.)
- [ ] CSP policy defined and tested

**Secrets Management**
- [ ] No secrets in version control
- [ ] Using `credentials.yml.enc` or environment variables

**Dependencies**
- [ ] All gems up to date
- [ ] Ran `bundle audit check`
- [ ] Ran `brakeman` scan

### Common Mistakes to Avoid

1. **Bypassing CSRF Protection**
   ```ruby
   # DON'T
   skip_before_action :verify_authenticity_token
   # Unless it's truly an API endpoint with other auth
   ```

2. **Exposing Sensitive Data in Logs**
   ```ruby
   # DO
   # config/initializers/filter_parameter_logging.rb
   Rails.application.config.filter_parameters += [
     :password, :password_confirmation, :token, :secret,
     :ssn, :credit_card, :cvv, :api_key
   ]
   ```

3. **Weak Session Secrets**
   ```ruby
   # DON'T: Short or predictable secret
   secret_key_base: "abc123"

   # DO: Generate with `rails secret`
   secret_key_base: "a4f2e8c9b5d7...long random string"
   ```

## Conclusion: Security as a Mindset

After diving deep into Rails security, here's what I've learned:

**Rails makes security easy, but not automatic.** You still have to follow the patterns:

1. **Use the framework's defaults** - They're there for good reason
2. **Never trust user input** - Validate, sanitize, escape
3. **Think like an attacker** - What could go wrong?
4. **Test security features** - Don't assume they work
5. **Keep dependencies updated** - New vulnerabilities are discovered daily
6. **Audit regularly** - Use tools like Brakeman and bundler-audit

The best part? Rails handles about 90% of security concerns automatically if you don't fight it. The other 10% is about understanding WHY these protections exist and using them correctly.

---

