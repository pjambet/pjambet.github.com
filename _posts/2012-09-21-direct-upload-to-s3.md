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
	  <p>Since beginning of september, Amazon added <a href="http://www.w3.org/TR/cors/">CORS</a> support to S3. As this is quite recent, there are not yet a lot of documentation and tutorial about how to set eveything up and running for your app.</p>
	  <p>Furthermore, <a href="http://blueimp.github.com/jQuery-File-Upload/">this jQuery plugin</a> is awesome, mainly for the progress bar handling, and the example <a href="https://github.com/blueimp/jQuery-File-Upload/wiki/Upload-directly-to-S3">in the wiki</a> is obsolete.</p>
    <p>If somehow you're working with heroku you might have already faced the 30s limit on each requests. There are some alternative, such as the extension of the great <a href="https://github.com/jnicklas/carrierwave">carrier wave gem</a>, <a href="https://github.com/dwilkie/carrierwave_direct">carrierwave direct</a>. I gave it a quick look, but I found it quite crappy, as it forces you to change your carrier wave settings (removing the store_dir method, really ?) globally and it only works for a single file. So I thought it would be better to handle upload manually for big files, and rely on vanilla carrier_wave for my other uploads.</p>
	  <p>Others example were interesting but they all lacked important things, and none of them worked for me out of the box, hence this short guide. This tutorial is inspired by <a href="http://highgroove.com/articles/2012/09/11/upload-directly-to-Amazon-s3-with-support-for-cors.html">that post</a> and <a href="http://www.ioncannon.net/programming/1539/direct-browser-uploading-amazon-s3-cors-fileapi-xhr2-and-signed-puts/">that one</a>.</p>
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
    <p>Of course those settings are only for development purpose, you'll probably want to restrict the Allowed Origin rule to your domain only. <a href="http://docs.amazonwebservices.com/AmazonS3/latest/dev/cors.html">Documentation</a> about those settings is quite good</p>

		<h2>Setup your server</h2>
		<p>In order to send your files to s3, you have to include a set of otions as described <a href="http://aws.amazon.com/articles/1434">in the official doc here</a> and <a href="">there</a></p>
		<p>One solution would be to directly write the content of all those variables in the form, so it's ready to be submitted, but I believe that most of those value should not be written in the DOM. So we'll create a new route we'll use to fetch those data</p>
    <p>This example is written with Rails, but writing the same for another framework should be really simple</p>
    {% highlight ruby %}
    MyApp::Application.routes.draw do
      resource signed_url, only: :index
    end
    {% endhighlight %}

    <p>Now that we have our new route, let's create the controller which will send back our data to the s3 form</p>

    {% highlight ruby %}
    SignedUrlsController < ApplicationController
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
              { bucket: MY_BUCKET },
              { acl: 'private' },
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
            AWS_SECRET_KEY_ID,
            s3_upload_policy_document
          )
        ).gsub(/\n/, '')
      end
    end
    {% endhighlight %}
    <p>So this is all we have to do on the server side</p>

		<h2>Add the jQueryFileUpload files</h2>
		<p>Next you'll have to have to add the jQueryFileUpload files. THe plugins ships with a lof of files, but I found a lot of them unecessary, so here is the list
		<ul>
		<li>vendor/jquery.ui.widget</li>
		<li>jquery.fileupload</li>
		</ul>
		</p>

		<h2>Setup the javascript client side</h2>
    <p>Now let's setup jQueryFileUpload to send the correct data to s3</p>
    <p>Based on what we did on the server, the workflow will be composed of 2 requests, first, it's going to fetch the needed data from our server, then send everything to

    <p>Here is the sample form I'm using</p>

    {% highlight haml %}
     %form#file_upload(action="https://BUCKET.s3.amazonaws.com" method="post" enctype="multipart/form-data")
        %input{type: :hidden, name: :key}
        %input{type: :hidden, name: "AWSAccessKeyId", value: AWS_ACCESS_KEY_ID}
        %input{type: :hidden, name: :acl, value: 'private'}
        %input{type: :hidden, name: :policy}
        %input{type: :hidden, name: :signature}
        %input{type: :hidden, name: :success_action_status, value: "201"}

        .progress.progress-striped.active
          .bar
        .file-upload.file-wrapper
          %legend
            %span.fileinput-button Track
          .custom-file-input
            %input.fake-input{type: 'text'}
            %button Choissisez un fichier
            %input#track-file{type: :file, name: :file}

    {% endhighlight %}

    {% highlight javascript %}

$(function() {

  $('#file-input').each(function() {

    var form = $(this).parents('form')

    $(this).fileupload({
      url: 'https://BUCKET.s3.amazonaws.com/',
      type: 'POST',
      autoUpload: true,
      dataType: 'xml', // This is really important as s3 gives us back the url of the file in a XML document
      add: function (event, data) {
        $.ajax({
          url: "/signed_urls",
          type: 'GET',
          dataType: 'json',
          data: {doc: {title: data.files[0].name}},
          async: false,
          success: function(retdata) {
            // after we created our document in rails, it is going to send back JSON of they key,
            // policy, and signature.  We will put these into our form before it gets submitted to amazon.
            form.find('input[name=key]').val(retdata.key)
            form.find('input[name=policy]').val(retdata.policy)
            form.find('input[name=signature]').val(retdata.signature)
          }

        })

        data.submit();
      },
      send: function(e, data) {
        $('.progress').fadeIn();
      },
      progress: function(e, data){
        var percent = Math.round((e.loaded / e.total) * 100)
        $('.bar').css('width', percent + '%')
      },
      fail: function(e, data) {
        console.log('fail')
      },
      success: function(data) {
        var url = $(data).find('Location').text()

        $('#track_track_url').val(url)
        $('input[type=submit]').attr('disabled', false)
      },
      done: function (event, data) {

        $('#track-file').siblings('.fake-input').val(data.files[0].name)
        $('.progress').fadeOut(300, function() {
          $('.bar').css('width', 0)
        })
      },
    })
  })
})


    {% endhighlight %}
    <p>So quick explanation about what's going on here</p>
    <p>In my example, this form sits next to a 'real' rails form, so once the upload is done I fill the 'real' field so my object is correctly created with the good url without having to handle extra things.</p>

	<h2>Live example</h2>
  <p>I'll set up a live example running on heroku, on which you'll be able to upload files in more than 30s coming soon </p>


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
			var disqus_shortname = 'pjambet'; // required: replace example with your forum shortname
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
