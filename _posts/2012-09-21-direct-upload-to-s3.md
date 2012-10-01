---
layout: post
title: Direct upload to s3 with cors
category: Coding
tags: aws javascript rails
year: 2012
month: 9
day: 21
published: true
summary: How to take full advantage of amazon s3 new CORS support
image: http://www.nasa.gov/images/content/686472main_pia16160-43_800-600.jpg
---

<div class="row">
	<div class="span9 columns">
	  <h2>Preface</h2>
	  <p>Since beginning of september, Amazon added <a href="http://www.w3.org/TR/cors/">CORS</a> support to S3. As this is quite recent, there are not yet a lot of documentation and tutorials about how to set eveything up and running for your app.</p>
	  <p>Furthermore, <a href="http://blueimp.github.com/jQuery-File-Upload/">this jQuery plugin</a> is awesome, mainly for the progress bar handling, but sadly the example <a href="https://github.com/blueimp/jQuery-File-Upload/wiki/Upload-directly-to-S3">in the wiki</a> is obsolete.</p>
    <p>If somehow you're working with heroku you might have already faced the 30s limit on each requests. There are some alternatives, such as the extension of the great <a href="https://github.com/jnicklas/carrierwave">carrier wave gem</a>, <a href="https://github.com/dwilkie/carrierwave_direct">carrierwave direct</a>. I gave it a quick look, but I found it quite crappy, as it forces you to change your carrier wave settings (removing the store_dir method, really ?) and it only works for a single file. So I thought it would be better to handle upload manually for big files, and rely on vanilla carrier_wave for my other small uploads.</p>
	  <p>I found other interesting examples but they all lacked important things, and none of them worked out of the box, hence this short guide. This tutorial is inspired by <a href="http://highgroove.com/articles/2012/09/11/upload-directly-to-Amazon-s3-with-support-for-cors.html">that post</a> and <a href="http://www.ioncannon.net/programming/1539/direct-browser-uploading-amazon-s3-cors-fileapi-xhr2-and-signed-puts/">that one</a>.</p>
		<h2>Setup your bucket</h2>
		<p>First you'll need to setup your bucket to enable CORS under certain conditions.</p>

    {% highlight xml %}
    <CORSConfiguration>
      <CORSRule>
        <AllowedOrigin>*</AllowedOrigin>
        <AllowedMethod>GET</AllowedMethod>
        <AllowedMethod>POST</AllowedMethod>
        <AllowedMethod>PUT</AllowedMethod>
        <AllowedHeader>*</AllowedHeader>
      </CORSRule>
    </CORSConfiguration>
    {% endhighlight %}
    <p>Of course those settings are only for development purpose, you'll probably want to restrict the Allowed Origin rule to your domain only. <a href="http://docs.amazonwebservices.com/AmazonS3/latest/dev/cors.html">Documentation</a> about those settings is quite good.</p>

		<h2>Setup your server</h2>
		<p>In order to send your files to s3, you have to include a set of options as described <a href="http://aws.amazon.com/articles/1434">in the official doc here</a> and <a href="">there</a></p>
		<p>One solution would be to directly write the content of all those variables in the form, so it's ready to be submitted, but I believe that most of those value should not be written in the DOM. So we'll create a new route we'll use to fetch those data.</p>
    <p>This example is written with Rails, but writing the same for another framework should be really simple</p>
    {% highlight ruby %}
    MyApp::Application.routes.draw do
      resources :signed_url, only: :index
    end
    {% endhighlight %}

    <p>Now that we have our new route, let's create the controller which will send back our data to the s3 form</p>

    {% highlight ruby %}
    class SignedUrlsController < ApplicationController
      def index
        render json: {
          policy: s3_upload_policy_document,
          signature: s3_upload_signature,
          key: "uploads/#{SecureRandom.uuid}/#{params[:doc][:title]}",
          success_action_redirect: board_url
        }
      end

      private

      # generate the policy document that amazon is expecting.
      def s3_upload_policy_document
        Base64.encode64(
          {
            expiration: 30.minutes.from_now.utc.strftime('%Y-%m-%dT%H:%M:%S.000Z'),
            conditions: [
              { bucket: ENV['S3_BUCKET'] },
              { acl: 'public-read' },
              ["starts-with", "$key", "uploads/"],
              { success_action_status: '201' }
            ]
          }.to_json
        ).gsub(/\n|\r/, '')
      end

      # sign our request by Base64 encoding the policy document.
      def s3_upload_signature
        Base64.encode64(
          OpenSSL::HMAC.digest(
            OpenSSL::Digest::Digest.new('sha1'),
            ENV['AWS_SECRET_KEY_ID'],
            s3_upload_policy_document
          )
        ).gsub(/\n/, '')
      end
    end
    {% endhighlight %}
    <p>The policy and signature method are stolen from the linked blog posts above with one exception, I had to include the "starts-width" constraint, otherwise s3 was yelling 403 to me.</p>
    <p>Everything else is quite straight forward, there's just a small detail to consider if you set the acl to 'private', but more on that later.</p>
    <p>One last detail, the key value is actually the path of your file on your bucket, so set it to whatever you want but be sure it matches the constraint you set in the policy. Here we're using <code>params[:doc][:file]</code> to read the name of the file we're about to upload. We'll see more about that when setting the javascript.</p>
    <p>That's basically everything we have to do on the server side</p>

		<h2>Add the jQueryFileUpload files</h2>
		<p>Next you'll have to add the <a href="http://blueimp.github.com/jQuery-File-Upload/">jQueryFileUpload</a> files. The plugins ships with a lof of files, but I found most of them useless, so here is the list
		<ul>
      <li><code>vendor/jquery.ui.widget</code></li>
      <li><code>jquery.fileupload</code></li>
		</ul>
		</p>

		<h2>Setup the javascript client side</h2>
    <p>Now let's setup jQueryFileUpload to send the correct data to s3</p>
    <p>Based on what we did on the server, the workflow will be composed of 2 requests, first, it's going to fetch the needed data from our server, then send everything to s3.</p>

    <p>Here is the form I'm using, the order of parameter is important.</p>

    {% highlight haml %}
     %form(action="https://ENV['S3_BUCKET'].s3.amazonaws.com" method="post" enctype="multipart/form-data")
        %input{type: :hidden, name: :key}
        %input{type: :hidden, name: "AWSAccessKeyId", value: ENV['AWS_ACCESS_KEY_ID']}
        %input{type: :hidden, name: :acl, value: 'public-read'}
        %input{type: :hidden, name: :policy}
        %input{type: :hidden, name: :signature}
        %input{type: :hidden, name: :success_action_status, value: "201"}

        %input{type: :file, name: :file, data: { :'direct-upload' => true } }
        - # You can recognize some bootstrap markup here :)
        .progress.progress-striped.active
          .bar
    {% endhighlight %}

    {% highlight javascript %}
$(function() {

  $('[data-direct-upload]').each(function() {

    var form = $(this).parents('form').first()

    $(this).fileupload({
      url: form.attr('action'),
      type: 'POST',
      autoUpload: true,
      dataType: 'xml', // This is really important as s3 gives us back the url of the file in a XML document
      add: function (event, data) {
        $.ajax({
          url: "/signed_urls",
          type: 'GET',
          dataType: 'json',
          data: {doc: {title: data.files[0].name}}, // send the file name to the server so it can generate the key param
          async: false,
          success: function(data) {
            // Now that we have our data, we update the form so it contains all
            // the needed data to sign the request
            form.find('input[name=key]').val(data.key)
            form.find('input[name=policy]').val(data.policy)
            form.find('input[name=signature]').val(data.signature)
          }
        })
        data.submit();
      },
      send: function(e, data) {
        $('.progress').fadeIn();
      },
      progress: function(e, data){
        // This is what makes everything really cool, thanks to that callback
        // you can now update the progress bar based on the upload progress
        var percent = Math.round((e.loaded / e.total) * 100)
        $('.bar').css('width', percent + '%')
      },
      fail: function(e, data) {
        console.log('fail')
      },
      success: function(data) {
        // Here we get the file url on s3 in an xml doc
        var url = $(data).find('Location').text()

        $('#real_file_url').val(url) // Update the real input in the other form
      },
      done: function (event, data) {
        $('.progress').fadeOut(300, function() {
          $('.bar').css('width', 0)
        })
      },
    })
  })
})
    {% endhighlight %}

    <p>So quick explanation about what's going on here : </p>
    <p>The <code>add</code> callback allows us to fetch the missing data before the upload. Once we have the data, we simply insert them in the form</p>
    <p>The <code>send</code> and <code>done</code> callbacks are only used for UX purpose, they show and hide the progress bar when needed. The real magic is the <code>progress</code> callback as it gives you the current progress of the upload in the event argument.</p>
    <p>In my example, this form sits next to a 'real' rails form which is used to save an object which has amongst its attributes a file_url, linked to the "big file" we just uploaded. So once the upload is done I fill the 'real' field so my object is correctly created with the good url without having to handle extra things. After submitting the real form my object is saved with the URL of the file uploaded on S3.</p>
    <p>If you're uploading public files, you're good to go, everything's perfect. But if you're uploading private file (this is set with the acl params), you still have a last thing to handle.</p>
    <p>Indeed the url itself is not enough, if you try accessing it, you'll face some ugly xml <a href="https://s3-eu-west-1.amazonaws.com/lpdc/glyphicons_003_user.png">like that</a>.
    The solution I used was to use the <a href="http://amazon.rubyforge.org/">aws gem</a> which provides a great method : <a href="http://amazon.rubyforge.org/doc/classes/AWS/S3/S3Object.html">AWS::S3Object#url_for</a>. With that method, you can get an authorized url for the desired duration with your bucket name and the key (the path of your file in the bucket) of your file</p>
    <p>So my custom url accessor looked something like this : </p>

    {% highlight ruby %}
  def url
    parent_url = super
    # If the url is nil, there's no need to look in the bucket for it
    return nil if parent_url.nil?

    # This will give you the last part of the URL, the 'key' params you need
    # but it's URL encoded, so you'll need to decode it
    object_key = parent_url.split(/\//).last
    AWS::S3::S3Object.url_for(
      CGI::unescape(object_key),
      ENV['S3_BUCKET'],
      use_ssl: true)
  end
    {% endhighlight %}
    <p>This involves some weird handling with the <code>CGI::unescape</code>, and there's probably a better way to achieve this, but this is one way to do it, and it works fine.</p>
	<h2>Live example</h2>
  <p>I'll set up a live example running on heroku, on which you'll be able to upload files in more than 30s coming soon </p>
  coming soon
  
  	<h2>EDIT</h2>
  	<p>I changed every access to AWS variables (BUCKET, SECRET_KEY and ACCESS_KEY) by using environment variables.
  	By doing so you don't have to put the variables directly in your files, but you just have to set correctly the variables :</p>
	
	{% highlight ruby %}
	export S3_BUCKET=<YOUR BUCKET>
  	export AWS_ACCESS_KEY_ID=<YOUR KEY>
  	export AWS_SECRET_KEY_ID=<YOUR SECRET KEY>
	{% endhighlight %}
	
	<p>When deploying on heroku you just have to set the variables with </p>
	
	{% highlight ruby %}
	heroku config:add AWS_ACCESS_KEY_ID=<YOUR KEY> --app <YOUR APP>
	{% endhighlight %}
	
	</div>
</div>

<div class="row">
	<div class="span9 column">
			<p class="pull-right">{% if page.previous.url %} <a href="{{page.previous.url}}" title="Previous Post: {{page.previous.title}}"><i class="icon-chevron-left"></i></a> 	{% endif %}   {% if page.next.url %} 	<a href="{{page.next.url}}" title="Next Post: {{page.next.title}}"><i class="icon-chevron-right"></i></a> 	{% endif %} </p>
	</div>
</div>

<div class="row">
    <div class="span9 columns">
		<h2>Comments Section</h2>
	    <p>Feel free to comment on the post but keep it clean and on topic.</p>
		<div id="disqus_thread"></div>
		<script type="text/javascript">
			/* * * CONFIGURATION VARIABLES: EDIT BEFORE PASTING INTO YOUR WEBPAGE * * */
			var disqus_shortname = 'githubpagepjambet'; // required: replace example with your forum shortname
			var disqus_identifier = '{{ page.url }}';
			var disqus_url = 'http://pjambet.github.com{{ page.url }}';

			/* * * DON'T EDIT BELOW THIS LINE * * */
			(function() {
				var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
				dsq.src = 'http://' + disqus_shortname + '.disqus.com/embed.js';
				(document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
			})();
		</script>
		<noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
		<a href="http://disqus.com" class="dsq-brlink">blog comments powered by <span class="logo-disqus">Disqus</span></a>
	</div>
</div>

<!-- Twitter -->
<script>!function(d,s,id){var js,fjs=d.getElementsByTagName(s)[0];if(!d.getElementById(id)){js=d.createElement(s);js.id=id;js.src="//platform.twitter.com/widgets.js";fjs.parentNode.insertBefore(js,fjs);}}(document,"script","twitter-wjs");</script>

<!-- Google + -->
<script type="text/javascript">
  (function() {
    var po = document.createElement('script'); po.type = 'text/javascript'; po.async = true;
    po.src = 'https://apis.google.com/js/plusone.js';
    var s = document.getElementsByTagName('script')[0]; s.parentNode.insertBefore(po, s);
  })();
</script>
