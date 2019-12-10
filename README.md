# railsforgotpassword
Guide to Rails Forgot Password Logic

##Backend
* Command line - generate password resets controller and new view:
```ruby
rails g controller password_resets new
```

* Go to routes.rb file:  
Delete:  
```ruby
get 'password_resets/new'  
```  
Add:  
```ruby
resources :password_resets
```   

* Go to routes.rb file:  
```ruby
//this one will be for the init or a password reset
  put 'password_resets/new', to: 'password_resets#new'
//I used the put for the change password
  resources :password_resets
```  

* Add methods to app/controllers/password_resets_controller.rb - will reset and send new password.  

```ruby
//this one will initialize the password change process
def new
    user = User.find_by_email(params[:email])
    user.send_password_reset if user
    render json: {message:'E-mail sent with password reset instructions.'}
  end
  def edit
    @user = User.find_by_password_reset_token!(params[:id])
  end
//this will change the password
  def update
    @user = User.find_by_password_reset_token!(params[:id])
    if @user.password_reset_sent_at < 2.hour.ago
      render json: {error:'Password reset has expired'}
    elsif @user.update(user_params)
      render json: { message:'Password has been reset!'}
    else
      render :edit
    end
  end
  
  private
    # Never trust parameters from the scary internet, only allow the white list through.
    def user_params
      params.require(:user).permit(:password)
    end
``` 
* command line - Add password_reset_token and password_reset_sent_at columns to user model. 
```ruby
rails g migration AddPasswordResetToUsers password_reset_token password_reset_sent_at:datetime
rails db:migrate
```  

* Add the following code to app/models/user.rb - this contains the send_password_reset function  

```ruby
def send_password_reset
  generate_token(:password_reset_token)
  self.password_reset_sent_at = Time.zone.now
  save!
  UserMailer.forgot_password(self).deliver# This sends an e-mail with a link for the user to reset the password
end

#This generates a random password reset token for the user
def generate_token(column)
  begin
    self[column] = SecureRandom.urlsafe_base64
  end while User.exists?(column => self[column])
end
```  

* Edit forgot_password method app/mailers/user_mailer.rb- this contains all the different mailer methods Final def password_reset should be:  
```ruby  
def forgot_password(user)
    @user = user
    @greeting = "Hi"
    @url=Rails.configuration.x.web_app_url
    mail to: user.email, :subject => 'GarageListings Reset password instructions'
  end
```  
* Add configuration
    * Development.rb  
    ```ruby Rails.application.configure do
    config.x.web_app_url="http://localhost:3001/resetpassword/"
  end  
  ```  
    * Production.rb  
    ```ruby   Rails.application.configure do
    config.x.web_app_url="http://garagelistings.surge.sh/resetpassword/"
  end
```  
* Replace what's currently in app/views/user_mailer/forgot_password.html.erb with what's below - contains link to reset password  

```ruby
<!DOCTYPE html>
<html>

<head>
  <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
</head>

<body>
<img src="http://garagelistings.surge.sh/gl.png" alt="">
<%= @greeting %> <%= @user.firstname %>, <br>

To reset your password, click the URL below: <br>

<a href='<%=@url%><%= @user.password_reset_token %>'>Reset</a> <!--This is the link to reset your password-->

If you did not request your password to be reset, just ignore this e-mail and your password will stay the same.
</body>

</html>
```  

## Front End  
* In src/services/api-helper.js 
 
```ruby
export const resetPassword = async (resetToken, data) => {
  const resp = await api.put(`/password_resets/${resetToken}`, data)
  return JSON.stringify(resp.data);
}
export const resetPasswordInit = async (data) => {
  const resp = await api.put(`/password_resets/new`, data)
  return JSON.stringify(resp.data);
}
```  
