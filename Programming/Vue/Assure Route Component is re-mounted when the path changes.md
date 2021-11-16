The easiest way is to add a `:key` property to `router-view`:
```javascript
<router-view :key="$route.fullPath"></router-view>
```
This works, because view computes the delta by changes in properties throughout the shadow DOM. Since the `router-view` only changes via tracking the URL, view never notices changes inside of it. By providing a `:key` property, there now is a noticable change.

## Alternatives

Since this solution applies to all routes simultaneously, you need to work around the problem when just applying this to a single component.
The easiest solution is to `watch` changes to the `$route.params` in your component, adapting it's internal state as needed:
```javascript
mounted () {
  this.getUser(this.$route.params.id);
},
methods: {
  getUser(id) {
    this.$http.get('/api/users/' + id)
      .then(response => {
        this.user = response.data
      })
  }
},
watch: {
  '$route.params.id'(newId, oldId) {
    this.getUser(newId)
  }
}
```