Introduction
============

This package will help you to do grammatically accurate translations in your Nette application (framework
version 2.2+).

Installation is currently possible using Composer, but in addition to the usual `"require"` you'll have
to add the repository like this:

	"repositories": [
		{
			"type": "git",
			"url": "git://github.com/czukowski/I18n_Plural.git"
		}
	],
	"require": {
		"czukowski/i18n": "dev-nette/master",
	}

Manual installation is always an option as well.

Translation files are actually Neon files and may be located virtually anywhere in your application.
The suggested directory is `app/i18n`, that'll be used for examples.

In your application code you'll need to make sure the i18n directory is known to the Configurator,
perhaps in `index.php` or bootstrap:

	$parameters['i18nDir'] = $parameters['appDir'].'/i18n'

Services configuration example (`'en-us'` would be your default language):

	services:
		# This is the Translation service.
		i18n:
			class: I18n\NetteTranslator('en-us')
			setup:
				- attach(@i18n.reader)
		# This is the Reader service that is the source of translation strings.
		# It is possible to attach multiple readers to the translator.
		i18n.reader: I18n\Reader\NeonReader(%i18nDir%)

Place your translations into the i18n directory, like so:

 * `fr.neon` - General French translations,
 * `fr/be.neon` - Belgium French translations that are different from general French,
 * `fr/ch.neon` - Swiss French translations that are different from general French.

If you request the translation for 'fr-CH' locale, it'll look in the `fr/ch.neon` first, and failing
that in the general `fr.neon`. If the translation wasn't found even there, the untranslated input string
is returned.

The translation data structure is very similar to what you're used to in Neon configuration:

	string: řetězec
	section:
		string: 'řetězec v podsekci'

Some of the Nette controls are ready for translations, you just need to set the translator instance
to them, for example (this is assuming you've named your `NetteTranslator` service 'i18n'):

	// Set translator to control (Nette\Forms\Controls\BaseControl):
	$control->setTranslator($this->context->getService('i18n'));
	// Set translator to form (Nette\Forms\Form):
	$form->setTranslator($this->context->getService('i18n'));
	// Set translator to template (Nette\Templating\Template):
	$template->setTranslator($this->context->getService('i18n'));

After setting the translator to the templates, you'll be able to use the translation macro:
`{_'translate this'}`. We'll get into the details on its usage later on.

Read on to learn what's the practical usage of sections, how to use plurals and more.

Translation contexts
====================

Many languages use different words or inflections depending on a lot of circumstances, while it isn't much
problem in English, we can find an example there, too: suppose you want to display a string, that looks like
this: "His/her name is _name_" and you know the name of a person and his or her gender. Suppose you have
a function named `__()`, that does your translations and accepts optional arguments for parameters
replacement (this will actually be your NetteTranslator's `translate()` function or the translation macro).
Then the most trivial would be to do this:

	echo __($gender == 'f' ? 'His' : 'Her').__('name is :name', array(':name' => $name));

Although you can probably see it's not flexible at all. This message doesn't have to begin with pronoun in
other languages. This is already better:

	echo __(':their name is :name', array(':name' => $name, ':their' => __($gender == 'f' ? 'His' : 'Her')));

But what if there is a language, that changes other words as well? That's where the contextual translation
comes in handy. Consider just this:

	echo __('Their name is :name', $gender);

For that to work, we have defined the translation key `Their name is :name` with 2 contexts - `f` and `m`:

	'Their name is :name':
		f: 'Her name is :name'
		m: 'His name is :name'

Example
-------

	foreach (array('aimee', 'bob') as $username)
	{
		$person = ORM::factory('profile')->find($username);
		echo __('Their name is :name', $person->gender, array(':name' => $person->name));
	}

	// Outputs:
	// Her name is Aimee
	// His name is Bob

Example
-------

Some languages distinguish grammatical genders in way more situations, than just pronouns. Also, we can't
tell what grammatical gender a certain word is in different languages, as it may be quite random. Now we see,
that we can't always specify the required form, as we did with the given names. In this case, we can think
of a context in another way, a context can be just an object we want it to be related to.

Let's take Russian for an example, although many others will have similar translation structure as well.

	Enabled:
		user: Включен
		role: Включена
		other: Включено

Somewhere else:

	echo __('Enabled', 'user');
	// Включен

Note the `other` key, that'll be used for any other context than `user` or `role`.

Plural inflections
==================

If you've ever been bothered by labels like "1 file(s)", there is a solution for you.

Nice people at CLDR have taken their time to compile plural rules for a large number of languages. This
module includes all these rules and a function, that converts any number into a proper context for that
language. The possible contexts are `zero`, `one`, `two`, `few`, `many` and `other`. Most languages will
only have 2-3 of these, and any of them will always have `other` context.

The rules are defined in [these classes](https://github.com/czukowski/I18n_Plural/tree/master/classes/I18n/Plural).
If you don't see your language immediately, try looking into One.php, Two.php and other generic names, they
aggregate a large number of languages, that share same rules. All the files include the rules in human
readable format and a list of languages they apply to.

It may be important to note, that the plural context must be numeric (`is_numeric` must return `TRUE` for that
value) in order to be tested against the language plural rules. Otherwise it'll look for the exact translation
key.

Example
-------

English:

	'You have :count messages':
		one: 'You have one message'        # 1 message
		other: 'You have :count messages'  # more messages

Czech:

	'You have :count messages':
		one: 'Máte jednu zprávu'      # 1 message
		few: 'Máte :count zprávy'     # 2 - 4 messages
		other: 'Máte :count zpráv'    # more messages

*Note:* before doing something like I did above (I've replaced :count with actual 'one' value for the
context `one`), check with the language rules, whether that context really applies only when the number
is 1. There are languages out there, where this is not the case, for those languages, you'll have to leave
the parameter there.

Example
-------

English:

	'Hi My Age Is':
		one: 'Hello world, I\'m :age year old'
		other: 'Hello world, I\'m :age years old'

Russian:

	'Hi My Age Is':
		one: 'Привет мир, мне уже :age год'
		few: 'Привет мир, мне уже :age года'
		many: 'Привет мир, мне уже :age лет'
		other: 'Привет мир, мне уже :age лет'

In your code:

	echo __('Hi My Age Is', 1, array(':age' => 1));
	// Hello world, I\'m 1 year old
	echo __('Hi My Age Is', 2, array(':age' => 2));
	// Hello world, I\'m 2 years old
	echo __('Hi My Age Is', 10, array(':age' => 10));
	// Hello world, I\'m 10 years old
	
	// Now suppose we've switched to another language
	
	echo __('Hi My Age Is', 1, array(':age' => 1));
	// Привет мир, мне уже 1 год
	echo __('Hi My Age Is', 2, array(':age' => 2));
	// Привет мир, мне уже 2 года
	echo __('Hi My Age Is', 10, array(':age' => 10));
	// Привет мир, мне уже 10 лет

Note how the 2nd and 3rd translations differ between the languages. For English, it's the same form ('years
old'), while in Russian the translations are totally different.

Translation models
==================

The concept of the translation models is simple: you have a phrase that you need to translate using different
parameters, contexts and languages. This is fairly easy to achieve using the core translation function, but
models allow you to move this logic to a separate class or object.

Example, of course, with the correct inflections:

	echo $filesInDirsCount->translate(123, 56, 'en');
	// Found 123 files in 56 directories
	echo $filesInDirsCount->translate(123, 56, 'ru');
	// Найдено 123 файла в 56 папках

You may implement models by taking off from various levels:

  1. By just implementing `I18n\Model\ModelInterface` which only requires your class to be castable to string
     using `__toString()` function. This means you're in the full control of how your model will be working.
  2. By extending `I18n\Model\ModelBase` that has various getter/setter methods to maintain model states,
     you'll need to implement the `translate()` function where you'll place the translation logic.
  3. By extending or even using directly the `I18n\Model\ParameterModel` where you only define the
     `translate()` function arguments types and default values and then use it just as in the example above.
     All arguments are optional, those not passed to the function will default to model states and failing
     that to the arguments' default values. Using this method may make it easier for common translation cases,
     but will lack in flexibility for more complex phrases. For more detailed description, look into the class
     itself for the code comments.

Also note that there are few sample models under the tests folder.

Translating in templates
========================

Here are some examples, those are pretty self-explanatory:

	// Basic translation.
	{_'Welcome!'}
	// Translation with context.
	{_'New customer has been saved.', $customer->gender}
	// Translation with parameters and context skipped.
	{_'Hi, my name is :name', [':name' => $name]}
	// All arguments present, including target language.
	{_'You have :count messages', $count, [':count' => $count], 'cs'}

API
===

### class I18n\Core

#### public function attach(I18n\Reader\ReaderInterface $reader)

  * @param  I18n\Reader\ReaderInterface  $reader

This method takes a class instance that implements `I18n\Reader\ReaderInterface`. `NetteReader` instance may
be used for Nette applications, that'll read the translations from the *.php files in your Nette `app/i18n`
dir; although you may implement your own readers to provide translations from any source of your choice.

#### public function translate($string, $context, $values, $lang = NULL)

 * @param   string  $string   String to translate
 * @param   mixed   $context  String form or numeric count
 * @param   array   $values   Param values to insert
 * @param   string  $lang     Target language (optional)
 * @return  string

Translation/internationalization function with context support. The PHP function
[strtr](http://php.net/strtr) is used for replacing parameters.

	$i18n->translate(':count user is online', 1000, array(':count' => 1000));
	// 1000 users are online

#### public function form($string, $form = NULL, $lang = NULL)

 * @param   string  $string  String to translate
 * @param   string  $form    String context form, if NULL, looking for 'other' form, else the very first form
 * @param   string  $lang    Target language (optional)
 * @return  string

Returns specified form of a string translation. If no translation exists, the original string will be
returned. No parameters are replaced.

	$hello = $i18n->form('I\'ve met :name, he is my friend now.', 'fem');
	// I've met :name, she is my friend now.

#### public function plural($string, $count = 0, $lang = NULL)

 * @param   string  $string  String to translate
 * @param   mixed   $count   Integer context form, 0 by default
 * @param   string  $lang    Target language (optional)
 * @return  string

Returns translation of a string. If no translation exists, the original string will be returned.
No parameters are replaced.

	$hello = $i18n->plural('Hello, my name is :name and I have :count friend.', 10);
	// 'Hello, my name is :name and I have :count friends.'

### interface I18n\Reader\ReaderInterface

The Reader must be able to return an associative array, if more than one translation option is available.
The 'other' key has a special meaning of a default translation.

#### public function get($string, $lang = NULL)

 * @param   string   text to translate
 * @param   string   target language
 * @return  mixed

Returns translation of a string or array of translation options. No parameters are replaced. It is up
to the implementation where it gets it.

Testing
=======

Although a golden rule says you aren't supposed to test the 3rd party code (that's its authors' responsibility),
you may run it using this command from module's root directory:

	phpunit --bootstrap tests/I18n/bootstrap.php tests/I18n/
