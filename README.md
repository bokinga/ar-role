ActiveRecord Variation Extension for Yii 2
==========================================

This extension provides support for ActiveRecord relation role (table inheritance) composition.

For license information check the [LICENSE](LICENSE.md)-file.

[![Latest Stable Version](https://poser.pugx.org/yii2tech/ar-role/v/stable.png)](https://packagist.org/packages/yii2tech/ar-role)
[![Total Downloads](https://poser.pugx.org/yii2tech/ar-role/downloads.png)](https://packagist.org/packages/yii2tech/ar-role)
[![Build Status](https://travis-ci.org/yii2tech/ar-role.svg?branch=master)](https://travis-ci.org/yii2tech/ar-role)


Installation
------------

The preferred way to install this extension is through [composer](http://getcomposer.org/download/).

Either run

```
php composer.phar require --prefer-dist yii2tech/ar-role
```

or add

```json
"yii2tech/ar-role": "*"
```

to the require section of your composer.json.


Usage
-----

This extension provides support for ActiveRecord relation role composition, which is also known as table inheritance.

For example: assume we have a database for the University. There are students studying in the University and there are
instructors teaching the students. Student has a study group and scholarship information, while instructor has a rank
and salary. However, both student and instructor have name, address, phone number and so on. Thus we can split
their data in the three different tables:
 - 'Human' - stores common data
 - 'Student' - stores student special data and reference to the 'Human' record
 - 'Instructor' - stores instructor special data and reference to the 'Human' record

DDL for such solution may look like following:

```sql
CREATE TABLE `Human`
(
   `id` integer NOT NULL AUTO_INCREMENT,
   `role` varchar(20) NOT NULL,
   `name` varchar(64) NOT NULL,
   `address` varchar(64) NOT NULL,
   `phone` varchar(20) NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE InnoDB;

CREATE TABLE `Student`
(
   `humanId` integer NOT NULL,
   `studyGroupId` integer NOT NULL,
   `hasScholarship` integer(1) NOT NULL,
    PRIMARY KEY (`humanId`)
    FOREIGN KEY (`humanId`) REFERENCES `Human` (`id`) ON DELETE CASCADE ON UPDATE CASCADE,
) ENGINE InnoDB;

CREATE TABLE `Instructor`
(
   `humanId` integer NOT NULL,
   `rankId` integer NOT NULL,
   `salary` integer NOT NULL,
    PRIMARY KEY (`humanId`)
    FOREIGN KEY (`humanId`) REFERENCES `Human` (`id`) ON DELETE CASCADE ON UPDATE CASCADE,
) ENGINE InnoDB;
```

This extension introduces [[\yii2tech\ar\role\RoleBehavior]] ActiveRecord behavior, which allows role relation based
ActiveRecord inheritance.
In oder to make it work, first of all, you should create an ActiveRecord class for the base table, in our example it
will be 'Human':

```php
class Human extends \yii\db\ActiveRecord
{
    public static function tableName()
    {
        return 'Human';
    }
}
```

Then you will be able to compose ActiveRecord classes, which implements role-based inheritance using [[\yii2tech\ar\role\RoleBehavior]].
There 2 different ways for such classes composition:
 - Master role inheritance
 - Slave role inheritance


## Master role inheritance <span id="master-role-inheritance"></span>

This approach assumes role ActiveRecord class be descendant of the base role class:

```php
class Student extends Human // extending `Human` - not `ActiveRecord`!
{
    public function behaviors()
    {
        return [
            'roleBehavior' => [
                'class' => RoleBehavior::className(), // Attach role behavior
                'roleRelation' => 'studentRole', // specify name of the relation to the slave table
            ],
        ];
    }

    public function getStudentRole()
    {
        // Here `StudentRole` is and ActiveRecord, which uses 'Student' table :
        return $this->hasOne(StudentRole::className(), ['humanId' => 'id']);
    }
}
```

The main benefit of this approach is that role class directly inherits all methods, validation and other logic from
the base one. However, you'll need to declare am extra ActiveRecord class, which corresponds the role table.
Yet another problem is you'll need to separate 'Student' records from 'Instructor' ones for the find process.
Without this following code, will return all 'Human' records, both 'Student' and 'Instructor':

```php
$students = Student::find()->all();
```

The solution for this could be introduction of special column 'role' in the 'Human' table and usage of the default
scope:

```php
class Student extends Human
{
    // ...

    public static find()
    {
        return parent::find()->where(['role' => 'student']);
    }
}
```

## Slave role inheritance <span id="slave-role-inheritance"></span>

This approach assumes role ActiveRecord does not extends the base one, but relates to it:

```php
class Instructor extends \yii\db\ActiveRecord // do not extending `Human`!
{
    public function behaviors()
    {
        return [
            'roleBehavior' => [
                'class' => RoleBehavior::className(), // Attach role behavior
                'roleRelation' => 'human', // specify name of the relation to the master table
                'isOwnerSlave' => true, // indicate that owner is a role slave - not master
            ],
        ];
    }

    public function getHuman()
    {
        return $this->hasOne(Human::className(), ['id' => 'humanId']);
    }
}
```

This approach does not require extra ActiveRecord class for functioning and it does not need default scope specification.
But it does not inherit logic declared in the base ActiveRecord.


## Accessing role attributes <span id="accessing-role-attributes"></span>

After being attached [[\yii2tech\ar\role\RoleBehavior]] provides access to the properties of the model bound by relation,
which is specified via [[\yii2tech\ar\role\RoleBehavior::roleRelation]], as they were the main one:

```php
$model = Student::findOne(1);
echo $model->studyGroupId; // equals to $model->studentRole->studyGroupId

$model = Instructor::findOne(2);
echo $model->name; // equals to $model->human->name
```

If the related model does not exist, for example, in case of new record, it will be automatically instantiated:

```php
$model = new Student();
$model->studyGroupId = 12;

$model = new Instructor();
$model->name = 'John Doe';
```
