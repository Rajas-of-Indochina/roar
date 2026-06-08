# Minority System Rewrite
The Minority System has been rewritten, and while it's still mostly the same, there's enough changes to merit a writeup on how to use it effectively.
## Data Structures
Each county has two sets of two variable lists used for Minorities. A Large and small list each for Faith minorities and Culture minorities:

- `faith_minorities_large`
- `faith_minorities_small`
- `culture_minoroties_large`
- `culture_minorities_small`

Each variable list contains a list of minorities, in the form of culture or faith scopes. Not all counties necessarily have these variable lists initialized, but the scripted effects are already set to check for that, so no need to worry.

However:

### do not directly manipulate these variable lists

We have scripted effects to interface with these lists. Messing with them manually in-script will just cause mess and headaches.

## Basic Minority Operations: `add` and `delete`
There's two pairs of basic operations for minorities, used in low-level effects to apply them to a specific minority list in a county. They are:

- `add_faith_minority_effect` and `delete_faith_minority_effect` for faiths
- `add_culture_minority_effect` and `delete_culture_minority_effect` for cultures

As one might expect, they respectively add or delete a minority. You aren't likely to be using these effects directly, but they're good to know how to use. They are called **from county scope**, as follows:
```
title:c_example = {
	add_faith_minority_effect = {
		FAITH = faith:example_faith
		SIZE = large
	}
	delete_culture_minority_effect = {
		CULTURE = culture:yankee
		SIZE = small
	}
}
```

As you can see, they take two parameters `CULTURE`/`FAITH` and `SIZE`. At present, the only sizes supported are `large` and `small` (**case sensitive**).

While these scripted effects are designed to not apply a minority if it's already present or is the county majority, they are still limited because you need to pre-specify a size, which can be inconvenient for wanting to make a small minority large or make a large minority small. Which is why we have the second-order scripted effects:

## Promotion and Demotion
`promote_faith_minority_effect`, `promote_culture_minority_effect`, `demote_faith_minority_effect`, and `demote_culture_minority_effect` ***are the primary effects for manipulating minorities in most scenarios.***

They are called from within county scope, and will automatically upgrade or downgrade a minority of a specified type in the county.

```
title:c_ejemplo = {
	promote_faith_minority_effect = {
		FAITH = faith:pastafarian
		# this will do nothing if the county is already majority of this faith
	}
	demote_culture_minority_effect = {
		CULTURE = culture:weeaboo
		# does nothing if the county is majority this culture and/or has no minority of it
	}
}
```

If you want to make a minority appear in a county, use `promote_`. If you want to convert a county, use `promote_` twice. Minority cleanup is already handled by the scripted effects.

## Expulsion and Migration
This is one of the more complex parts of this system, and so I'll break it down piece by piece, starting with the migration effects.

### The Actual Migration Effects
These are three scripted effects, `culture_migration_effect` (culture-only), `faith_migration_effect` (faith-only), or `culture_migration_effect` (combined culture-faith) migration. They have the parameters:

- `FAITH`: Used for faith and combined migrations. The faith that's migrating out.
- `CULTURE`: Used for culture and combined migrations. The culture that's migrating out.
- `TARGET_COUNTY`: Used for all three effects. The county that is being migrated TO.
- `SOURCE_COUNTY`: Used for all three effects. The county that is being migrated FROM.

Since they take counties as parameter inputs, they can be called without changing scope. Below is an example call of the effect `combined_migration_effect`:
```
combined_migration_effect = {
	CULTURE = culture:norman
	FAITH = faith:catholic
	SOURCE_COUNTY = title:c_evreux
	TARGET_COUNTY = title:c_london
}
```
This is equivalent to:
```
title:c_london = {
	promote_culture_minority_effect = { CULTURE = culture:norman }
	promote_faith_minority_effect = { FAITH = faith:catholic }
}
title:c_evreux = {
	demote_culture_minority_effect = { CULTURE = culture:norman }
	demote_faith_minority_effect = { FAITH = faith:catholic }
}
```
as well as sending a popup to the holders of each county informing them of the migration. If you want to model a direct county-to-county migration, these scripted effects are the simplest, easiest, cleanest way.

### Random Migration Targets
However, oftentimes, you can't or don't want to have a pre-determined target county for migration. This is where we use the effects `find_culture_migration_target_effect`, `find_faith_migration_target_effect` and `find_combined_migration_target_effect` effects to get a random valid target if one exists. They use the following parameters:

- `CULTURE`: Culture and Combined
- `FAITH`: Faith and Combined

This is meant to be called within the scope of a source county, and saves a scope, `scope:target_county` for later use.

Additionally, there's also ways to retrieve a random cultures or faiths for a migration: `get_random_county_faith_effect` and `get_random_county_culture_effect`, which are called from county scope and take the parameter `MAJORITY`, which uses `yes` or `no` to determine if it's allowed to include the majority in its random pick. They save the scopes `scope:faith` and `scope:culture` respectively.

Below is a series of calls to run a random migration from one of a character's counties automatically:
```
random_realm_county = {
	save_scope_as = source_county
	get_random_county_culture_effect = { MAJORITY = yes }
	get_random_county_faith_effect = { MAJORITY = yes }
	find_combined_migration_target_effect = {
		CULTURE = scope:culture
		FAITH = scope:faith
	}
	combined_migration_effect = {
		CULTURE = scope:culture
		FAITH = scope:faith
		SOURCE_COUNTY = scope:source_county
		TARGET_COUNTY = scope:target_county
	}
}
```
This compresses dozens of lines into just a couple effect calls.

### Expulsion
With all that explained, `expel_county_faith_minority_effect` and `expel_county_culture_minority_effect` become quite simple. It just takes `COUNTY`, the county that the minority is being expelled from, `FAITH`/`CULTURE`, the minority to be expelled, and `CHARACTER`, the character actually doing the expulsion, who receives some cash and tyranny based on the development of the county. Then it automatically finds an appropriate migration target and performs a migration, all in one go!

### Region-to-Region Migration
This is a quick, easy way to simulate a directed, targeted migration between two geographical regions, a `SOURCE_REGION` and a `TARGET_REGION`. Just one quick call:
```
quick_region_migration_effect = {
	SOURCE_REGION = world_europe_britannia
	TARGET_REGION = world_steppe_west
}
```
All handled instantly.

### Fake Migrations
There's also a `notify_characters_of_fake_migration_effect` which is only used to notify characters for fake migrations such as off-map culture/faith migrations from RICE. It expects a `scope:county` and it takes a `MESSAGE` parameter that is just the tooltip used for the notification.

## Adding Minorities in history
While Minorities can be added in on_action by simply calling the proper effects, it's preferable to assign minorities to counties in history. This can be done as follows:

```
#inside a \history\landed_titles file
c_examplestan = {
	866.2.3 = {
		effect = { add_faith_minority_effect = { FAITH = faith:placeholder SIZE = small } }
	}
	1055.2.1 = {
		# The minority got larger
		effect = {
			delete_faith_minority_effect = { FAITH = faith:placeholder SIZE = small }
			add_faith_minority_effect = { FAITH = faith:placeholder SIZE = large }
		}
	}
	1145.8.11 = {
		# they became a majority (set the majority in the provinces history)
		effect = {
			delete_faith_minority_effect = { FAITH = faith:placeholder SIZE = large } 
		}
	}
}
```
## Scripted Triggers

### `county_has_SIZE_TYPE_minority`:
Use these to determine if a county has any minority of a given size (`large`/`small`) and type (`faith`/`culture`):

- `county_has_large_faith_minority`
- `county_has_large_culture_minority`
- `county_has_small_faith_minority`
- `county_has_small_culture_minority`

### `county_has_TYPE_minority`:
Use these to detemine if a county has any minority of a given type (`faith`/`culture`) without respect to size:

- `county_has_faith_minority`
- `county_has_culture_minority`

### `county_has_specific_SIZE_TYPE_minority`:
Use these to determine if a county has a specific minority of a given size (`large`/`small`) and type (`faith`/`culture`). Takes either `FAITH` or `CULTURE` as appropriate.

- `county_has_specific_large_faith_minority`
- `county_has_specific_large_culture_minority`
- `county_has_specific_small_faith_minority`
- `county_has_specific_small_culture_minority`

### `county_has_specific_TYPE_minority`
As above, but without respect to size:

- `county_has_specific_faith_minority`
- `county_has_specific_culture_minority`

### county_has_type_minority_with_trigger
This one is deceptively complex. It takes four parameters:

- `TYPE`: The type of minority (`culture` or `faith`)
- `SIZE`: The size of minority (`large` or `small`)
- `TRIGGER`: This can be any simple trigger, that is, one that takes a single input value, including `yes`, `no`, or any variety of scopes or event targets.
- `CHECK`: The value you want the input to be. This has to be a single value or a valid target.

Below is one example of this trigger being put to use: Finding a large faith minority of a faith whose head of faith is the same as that of ROOT.
```
title:c_county = {
	county_has_type_minority_with_trigger = {
		TYPE = faith
		SIZE = large
		TRIGGER = religious_head
		CHECK = ROOT.faith.religious_head
	}
}
```
That effectively translates to:
```
title:c_county = {
	any_in_list = {
		variable = faith_minorities_large
		religious_head = ROOT.faith.religious_head
	}
}
```
There's also a sister trigger, `county_has_type_minority_trigger_not` that works identically, but checks if the given `TRIGGER` and `CHECK` params evaluate to false.
