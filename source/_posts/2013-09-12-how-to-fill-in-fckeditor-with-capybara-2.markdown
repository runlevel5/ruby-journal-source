---
layout: post
title: "How to fill in FCKeditor with capybara 2"
date: 2013-09-12 12:46
comments: true
tags: rails capybara ruby
author: "Trung Lê"
---

{{ post.title }}

In this short tutorial, I will show you how to fill in FCkeditor text area by using Capybara 2

<!--more-->

Just imagine you have a textarea:

```
<textarea cols=​"40" id=​"letter_content" name=​"letter[content]​" rows=​"20">​</textarea>
```

Once you apply FCKeditor for this textarea, the output yields following new DOM:

```
<textarea cols=​"40" id=​"letter_content" name=​"letter[content]​" rows=​"20">​</textarea>
<div id="cke_letter_content" class="cke_1 cke cke_reset cke_chrome cke_editor_letter_content cke_ltr cke_browser_webkit" dir="ltr" lang="en" role="application" aria-labelledby="cke_letter_content_arialbl">
  <iframe src="" frameborder="0" class="cke_wysiwyg_frame cke_reset" style="width: 100%; height: 100%;" title="Rich Text Editor, letter_content" aria-describedby="cke_30" tabindex="0" allowtransparency="true">
    <body contenteditable="true" class="cke_editable cke_editable_themed cke_contents_ltr cke_show_borders" spellcheck="false">
    </body>
  </iframe>
</div>
```

As you can see in the DOM structure, FCKeditor will create new iframe and within this frame, we have a body tag with HTML5 contenteditable.
By common sense, in order to fill in this body tag, I'd do following:

* Switch to that editor frame
* Set the body content

I sit down and give it a try on caypybara + selenium and capybara + polstergeist:

```ruby
  # WARNNING: not working!!
  def fckeditor_fill_in(frame_id, value)
    within_frame(frame_id) do
      fill_in 'body', with: value
    end
  end
```

to use it:

```ruby
  fckeditor_fill_in('#cke_letter_content iframe:nth-of-type(1)', 'Hello world')
```

run the test and BOOM!, it failed. Firstly, `within_frame` does not work with selenium. Secondly, `fill_in` does not support edit contenteditable tag, there was a PR by Jon Rowe few months ago but not merged yet (as of this writing). It seems to me the only viable
solution left for me is to do it the monkey way, that is Javascript. And below is the working code:

```ruby
  def fckeditor_fill_in(id, params = {})
    page.execute_script %Q{
      var ckeditor = CKEDITOR.instances.#{id}
      ckeditor.setData('#{params[:with]}')
      ckeditor.focus()
      ckeditor.updateElement()
    }
  end
```

to use it:

```ruby
  fckeditor_fill_in('letter_content', 'Hello world')
```

Vioala! It works smoothly. The lesson I learned with capybara is that it is still a pain to use the built-in DSL as it doesn't work across
drivers. If you get stuck, fallback to the primitive way. Feedbacks are greatly welcomed.
