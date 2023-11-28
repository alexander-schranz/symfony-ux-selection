# Symfony UX Selection

A Symfony UX Form type for any kind of selection.

## Target of the project

Add a selection form type which is flexible enough and target some issues I currently having with the existing
[symfony/ux-autocomplete](https://github.com/symfony/ux-autocomplete).

 - Flexible enough to be used for different UI components via `skin` configuration:
    - `autocomplete`
    - `list` (table)
    - `list_overlay`
    - ... similar how current Choice can be Radio or Select but more extended and configurable.
 - Display multiple values not only a single value, depend on the `skin_options`:
 - Search is only one kind of filters other filters should also be supported.
 - Not connected to doctrine:
    - We should not care where the data for such selection is coming from.
      Whatever its our database, Elasticsearch, external API, ...
 - Support for `options` given from outside and not supported by EntityType
    - Current autocomplete errors if you give options which are not supported by EntityType as all options are forward
      but are even lost by the Controller.

## FormType usage

This currently should more show how such `SelectionType` could look like:

```php
class AnyField extends AbstractType
{
    public function __construct(
        private HttpClientInterface $httpClient,
    ) {
    }

    public function configureOptions(OptionsResolver $resolver)
    {
        $resolver->setDefaults([
            // search should not be the only filter so any form type should be able to use as a filter:
            'filter' => \Symfony\Component\Form\Extension\Core\Type\SearchType::class
            'filter_options' => [],
            // every skin should support single and multi selections
            'multiple' => true,
            // choices can be loaded from everywhere not only doctrine
            'choices' => function(array $filters = [], ?array $values = null): Result {
                if ($values !== null) { // when values are given the filter is always an empty array and we just need to filter by ids
                    $filters['ids'] = $values;
                }

                $result = $this->httpClient->request('GET', 'https://127.0.0.1:8080/api/choices', [
                    'query' => $filters,
                ]);
                $choices = $result['items'];
                $total = $result['total'];
                $limit = $result['limit'];

                return new Result(
                    $choices,
                    // a search form could also support pagination in its selection not sure yet how to handle this yet:
                    total: 100, 
                    limit: 10,
                );
            },
            'choice_value' => fn($choice) => $choice['id'],
            // maybe lazy should be the none configurable default so form always render fast and load data lazy
            'lazy' => true,
            // support for different skins
            'skin' => ChoiceType::class, 
            'skin_options' => [
                // 'multiple' => true, // is always forwarded and not need to be set
                // 'label' => false, // is by default false
                // will get the selected 'choices'
                'expanded' => true,
            ], 

            // custom skin
            'skin' => ListType::class, // TODO search result can be rendered different selected items so there need some kind of different between them
            'skin_options' => [
                'columns' => [
                    'name' => [
                        'label' => 'app.description',
                        'type' => 'text',
                    ],
                    'description' => [
                        'label' => 'app.description',
                        'type' => 'text',
                    ],
                ],
            ],
        ]);
        
        $resolver->addNormalizer('skin_options', function (Options $options, $value) {
             // maybe define different skin_options default values based on current given "skin"
            if ($options['skin'] === AutoCompleteType::class) {
                if (!isset($value['columns'])) {
                    $value['columns'] = ['name', 'description'];
                }

                return $value;
            }
            
            return $value;
        });
    }
    
    public function getParent(): string
    {
        return SelectionType::class;
    }
}
```
